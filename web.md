## Web Service Specifics

This document contains Web-specific code and configuration that differs from the shared content in `SKILL.md`.

---

### Step 1: Database Selection (Web)

Database is optional for Web services, same as CLI. Ask the user:

> "Does this project need a database?"
> - **None**: No database needed
> - **SQLite**: Lightweight embedded database, suitable for small projects and rapid prototyping
> - **DolphinDB**: High-performance time-series database, suitable for big data analytics and time-series data processing

If the user chooses **None**, skip all database-related steps.

### Step 2: Directory Structure (Web)

```
project/
├── app/
│   ├── api/
│   │   ├── api.py                  # InternalApi and ExternalApi base classes for HTTP API clients
│   │   ├── common.py              # API common utilities and constants
│   │   ├── {Name}Api.py            # API client classes for accessing external interfaces
│   │   └── __init__.py             # Package initialization
│   ├── common.py                   # Application-level common utilities and constants
│   ├── db/
│   │   ├── common.py               # Database common utilities and constants
│   │   ├── DB.py                   # Abstract database base class
│   │   ├── {Name}DB.py             # Database implementation classes
│   │   └── __init__.py             # Package initialization
│   ├── feature/
│   │   ├── {module}/
│   │   │   ├── common.py           # Feature-specific Errc enum, constants, enums, and public methods
│   │   │   ├── field.py            # Feature-specific Field classes (database table classes)
│   │   │   ├── dao.py              # Feature-specific DAO classes
│   │   │   ├── service.py          # Feature-specific Service classes
│   │   │   ├── handler.py          # Feature-specific Handler classes (web application only)
│   │   │   ├── task.py (optional)  # Feature-specific scheduled task classes
│   │   │   └── __init__.py         # Package initialization, exports all API client classes
│   ├── main.py                     # Application entry point
│   ├── middleware.py               # Middleware for request/response processing (web application only)
│   ├── task.py                     # Application-level scheduled tasks
│   └── __init__.py                 # Package initialization
├── config/
│   ├── config.toml                 # Production configuration file
│   └── config.dev.toml             # Development configuration file
├── data/                           # Data files
├── doc/
│   └── API.md                      # API documentation (web application only)
├── log/
│   └── main.log                    # Application log file (auto-generated at runtime)
├── test/                           # Test files
├── pyproject.toml                  # Poetry project configuration
├── poetry.lock                     # Poetry dependency lock file (auto-generated)
└── poetry.toml                     # Poetry global configuration
```

### Step 4: Configuration Files (Web)

#### config/config.dev.toml

```toml
# Log configuration
[log]
main = "DEBUG"  # Log level, supports DEBUG, INFO, WARNING, ERROR, CRITICAL

# Web service configuration
[web]
host = "0.0.0.0"  # Listen address, 0.0.0.0 means accept connections from all sources
port = 4002  # Listen port
body_limit = 10485760  # Request body size limit (bytes)
cors = true  # Whether to enable CORS cross-origin support

Only add the following database configuration if the user chose a database in Step 1:

```toml
# Database configuration
[service.db.sqlitedb]
path = "./data/main.db"  # SQLite database file path
timeout_s = 5.0  # Database operation timeout (seconds)
```

#### config/config.toml

```toml
# Log configuration
[log]
main = "INFO"  # Log level, supports DEBUG, INFO, WARNING, ERROR, CRITICAL

# Web service configuration
[web]
host = "0.0.0.0"  # Listen address, 0.0.0.0 means accept connections from all sources
port = 4002  # Listen port
body_limit = 10485760  # Request body size limit (bytes)
cors = false  # Whether to enable CORS cross-origin support
```

Only add the following database configuration if the user chose a database in Step 1:

```toml
# Database configuration
[service.db.sqlitedb]
path = "./data/main.db"  # SQLite database file path
timeout_s = 30.0  # Database operation timeout (seconds)
```

### Step 5: app/common.py Errc Enum (Web)

> Note: `Error`, `SuccessResponse`, `ErrorResponse`, `Pagination` classes are shared and defined in `SKILL.md` Step 5. Only the `Errc` enum is Web-specific.

```python
class Errc(Enum):
    """Common error code enum"""

    # HTTP related errors
    UNKNOWN_ERROR = "zimu::common::000"
    RESOURCE_NOT_FOUND = "zimu::common::001"
    METHOD_NOT_ALLOWED = "zimu::common::002"

    # Common errors
    INTERNAL_SERVER_ERROR = "zimu::common::003"
    BAD_REQUEST = "zimu::common::004"
    UNAUTHORIZED = "zimu::common::005"
    FORBIDDEN = "zimu::common::006"

    # Timeout errors
    TIMEOUT = "zimu::common::007"

    # INVALID errors
    INVALID_PARAMETER = "zimu::common::008"
    INVALID_JSON = "zimu::common::009"
    INVALID_TYPE = "zimu::common::010"

    # MISSING errors
    MISSING_PARAMETER = "zimu::common::011"
    MISSING_CONFIG = "zimu::common::012"

    def __str__(self) -> str:
        """Return error code string"""
        return self.value
```

### Step 6: Create app/middleware.py (Web Only)

Create `app/middleware.py` with the following content:

```python
"""Middleware module: unified management of all middleware"""

import logging
import time

from aiohttp import web

from app.common import Errc, Error, ErrorResponse

logger = logging.getLogger(__name__)

@web.middleware
async def error_middleware(request: web.Request, handler):
    """Error handling middleware

    Args:
        request: HTTP request
        handler: Request handler

    Returns:
        HTTP response
    """
    try:
        return await handler(request)
    except Error as e:
        # Business error
        message = f'failed to handle business error with path={request.path}, code={e.code}, message={e.message}'
        logger.error(message)
        logger.exception(e)
        return web.json_response(
            ErrorResponse(code=str(e.code), message=e.message).to_dict(),
            status=400
        )
    except web.HTTPException as e:
        # aiohttp built-in exceptions (404, 405, etc.)
        message = f'failed to handle http exception with path={request.path}, status={e.status}'
        logger.error(message)
        logger.exception(e)
        return web.json_response(
            ErrorResponse(
                code=str(Errc.INTERNAL_SERVER_ERROR),
                message=message
            ).to_dict(),
            status=e.status
        )
    except Exception as e:
        # Unexpected exception
        message = f'failed to handle request with path={request.path}'
        logger.error(message)
        logger.exception(e)
        return web.json_response(
            ErrorResponse(
                code=str(Errc.INTERNAL_SERVER_ERROR),
                message='internal server error'
            ).to_dict(),
            status=500
        )

@web.middleware
async def logging_middleware(request: web.Request, handler):
    """Logging middleware

    Args:
        request: HTTP request
        handler: Request handler

    Returns:
        HTTP response
    """
    # Record request start time
    start_time = time.perf_counter()

    # Log request info
    logger.debug(f'{request.method} {request.path}')

    # Call next middleware or handler
    response = await handler(request)

    # Calculate execution time (seconds, three decimal places for milliseconds)
    elapsed_s = time.perf_counter() - start_time
    elapsed_formatted = f'{elapsed_s:.3f}s'

    # Log response status and execution time
    logger.info(f'{request.method} {request.path} - {response.status} - {elapsed_formatted}')

    return response


@web.middleware
async def cors_middleware(request: web.Request, handler):
    """CORS middleware (open by default)

    Args:
        request: HTTP request
        handler: Request handler

    Returns:
        HTTP response
    """
    # Handle OPTIONS preflight request
    if request.method == 'OPTIONS':
        response = web.Response(status=200)
        response.headers['Access-Control-Allow-Origin'] = '*'
        response.headers['Access-Control-Allow-Methods'] = 'GET, POST, PUT, DELETE, OPTIONS, PATCH'
        response.headers['Access-Control-Allow-Headers'] = 'Content-Type, Authorization'
        response.headers['Access-Control-Max-Age'] = '86400'
        return response

    # Call next middleware or handler
    response = await handler(request)

    # Add CORS headers (open by default)
    response.headers['Access-Control-Allow-Origin'] = '*'
    response.headers['Access-Control-Allow-Methods'] = 'GET, POST, PUT, DELETE, OPTIONS, PATCH'
    response.headers['Access-Control-Allow-Headers'] = 'Content-Type, Authorization'

    return response
```

### Step 10: app/main.py (Web)

Create `app/main.py` with the following content:

```python
import os
import logging
import tomllib

from aiohttp import web

from app.middleware import cors_middleware, error_middleware, logging_middleware


def main():
    """Main function - create application, register routes, configure dependencies"""

    # Load configuration file
    env = os.getenv("APP_ENV", "dev")
    if env == 'production':
        config_path = 'config/config.toml'
    else:
        config_path = 'config/config.dev.toml'

    with open(config_path, "rb") as f:
        config = tomllib.load(f)

    # Optional: Only import if the user chose a database in Step 1
    # from app.db.SqliteDB import SqliteDB

    # Initialize logging
    log_level = config["log"]["main"]
    logging.basicConfig(
        level=getattr(logging, log_level),
        format='%(asctime)s.%(msecs)03d - %(name)s - %(levelname)s - %(message)s',
        datefmt='%Y-%m-%d %H:%M:%S'
    )

    # Initialize logger (after logging.basicConfig)
    logger = logging.getLogger(__name__)

    # Extract common configuration
    host = config["web"]["host"]
    port = config["web"]["port"]

    async def hello(request: web.Request) -> web.Response:
        """Health check endpoint"""
        return web.Response(text='hello')

    async def on_startup(app: web.Application):
        """Application startup hook - only handles async resource initialization"""
        logger.info(f'ready to startup application on host={host}, port={port}')

        sqlite_db = app.get("sqlite_db")
        if sqlite_db:
            await sqlite_db.connect()

        logger.info(f'succeeded to startup application on host={host}, port={port}')

    async def on_cleanup(app: web.Application):
        """Application cleanup hook"""
        logger.info(f'ready to cleanup application on host={host}, port={port}')

        sqlite_db = app.get("sqlite_db")
        if sqlite_db:
            await sqlite_db.close()

        logger.info(f'succeeded to cleanup application on host={host}, port={port}')

    # Optional: Database initialization (only if user chose a database in Step 1)
    # db_config = {
    #     "path": config["service"]["db"]["path"],
    #     "check_same_thread": True,
    #     "timeout": config["service"]["db"]["timeout_s"],
    #     "isolation_level": None
    # }
    # sqlite_db = SqliteDB(config=db_config)

    cors_enabled = bool(config["web"].get("cors", False))

    if cors_enabled:
        app = web.Application(middlewares=[logging_middleware, error_middleware, cors_middleware])
        logger.info(f'CORS middleware enabled with cors_enabled={cors_enabled}')
    else:
        app = web.Application(middlewares=[logging_middleware, error_middleware])
        logger.info(f'CORS middleware disabled with cors_enabled={cors_enabled}')

    # Optional: Store database instance (only if user chose a database)
    # app["sqlite_db"] = sqlite_db

    app.router.add_get("/hello", hello)

    app.on_startup.append(on_startup)
    app.on_cleanup.append(on_cleanup)

    logger.info(f'ready to run app on host={host}, port={port}')
    web.run_app(app, host=host, port=port)


if __name__ == "__main__":
    main()
```

---

## Feature Creation Process (Web)

### Step 4: Define Error Codes (common.py) (Web Only)

Create `app/feature/{name}/common.py` with the `Errc` enum class:

```python
from enum import Enum

class Errc(Enum):
    """Error code enum

    Error code format: myapp::module::number
    """
    UNKNOWN_ERROR = "myapp::user::000"  # Unknown error
    NOT_FOUND = "myapp::user::001"      # Resource not found
    USERNAME_EXISTS = "myapp::user::002"  # Username already exists
    # ... add more business error codes
```

### Step 5: Initialize Feature in app/main.py (Web)

According to the module dependency relationships described in `details/*.md`, initialize feature-related modules in `app/main.py` in order.

#### Initialization Order

Feature module initialization follows the dependency injection pattern, in the following order:

1. **DAO** (optional, only when database is needed) → 2. **Service** → 3. **Handler** → 4. **Register Routes**

#### Code Modification Location Example

The following uses the `user` feature as an example to show modification locations in `app/main.py`:

##### 1. Import Section (top of file)

Add feature-related imports after existing import statements:

```python
import os
import logging
import tomllib

from aiohttp import web

from app.middleware import error_middleware, logging_middleware, cors_middleware
from app.db.SqliteDB import SqliteDB
# === Add feature-related imports ===
from app.feature.user import UserDao, UserService, UserHandler
```

##### 2. Initialization Section (inside main function, after database initialization)

Add after database initialization (if any), before `app = web.Application(...)`:

```python
    # Optional: Database initialization (only if user chose a database in Step 1)
    # db_config = {
    #     "path": config["service"]["db"]["path"],
    #     "check_same_thread": True,
    #     "timeout": config["service"]["db"]["timeout_s"],
    #     "isolation_level": None
    # }
    # sqlite_db = SqliteDB(config=db_config)

    # === Initialize user feature ===
    # Optional: Only when database is needed
    # user_dao = UserDao(config=config)
    # user_dao.set_db(db=sqlite_db)

    user_service = UserService(config=config)
    # user_service.set_user_dao(dao=user_dao)

    user_handler = UserHandler(config=config)
    user_handler.set_user_service(service=user_service)

    cors_enabled = bool(config["web"].get("cors", False))

    if cors_enabled:
        app = web.Application(middlewares=[logging_middleware, error_middleware, cors_middleware])
        # ...
```

##### 3. Store in app Dictionary (optional, for lifecycle management)

Add after `app["sqlite_db"] = sqlite_db`:

```python
    app["sqlite_db"] = sqlite_db
    # === Store feature-related instances (optional, only when database is needed) ===
    # app["user_dao"] = user_dao
```

##### 4. Register Routes (route registration section)

Add feature routes after existing route registrations:

```python
    app.router.add_get("/hello", hello)

    # === Register feature routes ===
    user_handler.register_routes(app)

    app.on_startup.append(on_startup)
```

##### 5. Lifecycle Hooks (inside on_startup function, optional)

If the feature needs to initialize data tables at startup, add to the `on_startup` function:

```python
    async def on_startup(app: web.Application):
        """Application startup hook - only handles async resource initialization"""
        logger.info(f'ready to startup application on host={host}, port={port}')

        sqlite_db = app.get("sqlite_db")
        if sqlite_db:
            await sqlite_db.connect()

        # === Initialize feature data tables (only when database is needed) ===
        # user_dao = app.get("user_dao")
        # if user_dao:
        #     await user_dao.init_table()

        logger.info(f'succeeded to startup application on host={host}, port={port}')
```

---

## API Class Creation Process (Web)

### Step 6: Initialize API Class in app/main.py (Web)

Initialize the API class in `app/main.py`, modify at the following locations:

#### 1. Import Section (top of file)

```python
# Optional: Only import if the user chose a database in Step 1
# from app.db.SqliteDB import SqliteDB
# === Add API-related imports ===
from app.api import {Name}Api
```

#### 2. Initialization Section (inside main function, after database initialization)

Add after database initialization (if any), before `app = web.Application(...)`:

```python
    # Optional: Database initialization (only if user chose a database in Step 1)
    # sqlite_db = SqliteDB(config=db_config)

    # === Initialize API client ===
    {name}_api = {Name}Api(config=config)

    cors_enabled = bool(config["web"].get("cors", False))
```

#### 3. Lifecycle Hooks (inside on_cleanup function)

Add API session cleanup in the `on_cleanup` function:

```python
    async def on_cleanup(app: web.Application):
        """Application cleanup hook"""
        logger.info(f'ready to cleanup application on host={host}, port={port}')

        sqlite_db = app.get("sqlite_db")
        if sqlite_db:
            await sqlite_db.close()

        # === Close API session ===
        {name}_api = app.get("{name}_api")
        if {name}_api:
            await {name}_api.close()

        logger.info(f'succeeded to cleanup application on host={host}, port={port}')
```

#### 4. Store in app Dictionary

Add after `# app["sqlite_db"] = sqlite_db`:

```python
    # Optional: app["sqlite_db"] = sqlite_db
    # === Store API client instance ===
    app["{name}_api"] = {name}_api
```

#### 5: Use in Service (optional)

If the API client needs to be used by a feature's service, inject it during feature initialization:

```python
    # === Initialize feature and inject API client ===
    # Optional: Only when database is needed
    # my_dao = MyDao(config=config)
    # my_dao.set_db(db=sqlite_db)

    my_service = MyService(config=config)
    my_service.set_my_dao(dao=my_dao)
    my_service.set_{name}_api(api={name}_api)  # Inject API client
```
