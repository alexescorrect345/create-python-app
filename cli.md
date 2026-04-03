## CLI Program Specifics

This document contains CLI-specific code and configuration that differs from the shared content in `SKILL.md`.

---

### Step 1: Database Selection (CLI)

Database is optional for CLI programs. Ask the user:

> "Does this project need a database?"
> - **None**: No database needed
> - **SQLite**: Lightweight embedded database, suitable for small projects and rapid prototyping
> - **DolphinDB**: High-performance time-series database, suitable for big data analytics and time-series data processing

If the user chooses **None**, skip all database-related steps (Step 8, Step 9, database-related configuration, etc.).

### Step 2: Directory Structure (CLI)

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
│   │   │   ├── task.py (optional)  # Feature-specific scheduled task classes
│   │   │   └── __init__.py         # Package initialization, exports all API client classes
│   ├── main.py                     # Application entry point
│   ├── task.py                     # Application-level scheduled tasks
│   └── __init__.py                 # Package initialization
├── config/
│   ├── config.toml                 # Production configuration file
│   └── config.dev.toml             # Development configuration file
├── data/                           # Data files
├── log/
│   └── main.log                    # Application log file (auto-generated at runtime)
├── test/                           # Test files
├── pyproject.toml                  # Poetry project configuration
├── poetry.lock                     # Poetry dependency lock file (auto-generated)
└── poetry.toml                     # Poetry global configuration
```

### Step 4: Configuration Files (CLI)

#### config/config.dev.toml

```toml
# Log configuration
[log]
main = "DEBUG"  # Log level, supports DEBUG, INFO, WARNING, ERROR, CRITICAL
```

Only add the following database configuration if the user chose a database in Step 1:

```toml
# Database configuration
[service.db.sqlite]
path = "./data/main.db"  # SQLite database file path
timeout_s = 5.0  # Database operation timeout (seconds)
```

#### config/config.toml

```toml
# Log configuration
[log]
main = "INFO"  # Log level, supports DEBUG, INFO, WARNING, ERROR, CRITICAL
```

Only add the following database configuration if the user chose a database in Step 1:

```toml
# Database configuration
[service.db.sqlite]
path = "./data/main.db"  # SQLite database file path
timeout_s = 30.0  # Database operation timeout (seconds)
```

### Step 5: app/common.py Errc Enum (CLI)

> Note: `Error`, `SuccessResponse`, `ErrorResponse`, `Pagination` classes are shared and defined in `SKILL.md` Step 5. Only the `Errc` enum differs.

```python
class Errc(Enum):
    """Common error code enum"""

    # HTTP related errors
    UNKNOWN_ERROR = "zimu::common::000"
    RESOURCE_NOT_FOUND = "zimu::common::001"
    METHOD_NOT_ALLOWED = "zimu::common::002"

    # Timeout errors
    TIMEOUT = "zimu::common::003"

    # INVALID errors
    INVALID_PARAMETER = "zimu::common::004"
    INVALID_JSON = "zimu::common::005"
    INVALID_TYPE = "zimu::common::006"

    # MISSING errors
    MISSING_PARAMETER = "zimu::common::007"
    MISSING_CONFIG = "zimu::common::008"
    MISSING_DAO = "zimu::common::009"
    MISSING_SERVICE = "zimu::common::010"
    MISSING_FIELD = "zimu::common::011"

    def __str__(self) -> str:
        """Return error code string"""
        return self.value
```

### Step 10: app/main.py (CLI)

Create `app/main.py` with the following content:

```python
import os
import asyncio
import logging
import tomllib

# Optional: Only import if the user chose a database in Step 1
# from app.db.SqliteDB import SqliteDB

async def main():
    """Main function - create application, configure dependencies"""

    # Load configuration file
    env = os.getenv("APP_ENV", "dev")
    if env == 'production':
        config_path = 'config/config.toml'
    else:
        config_path = 'config/config.dev.toml'

    with open(config_path, "rb") as f:
        config = tomllib.load(f)

    # Initialize logging
    log_level = config["log"]["main"]
    logging.basicConfig(
        level=getattr(logging, log_level),
        format='%(asctime)s.%(msecs)03d - %(name)s - %(levelname)s - %(message)s',
        datefmt='%Y-%m-%d %H:%M:%S'
    )

    # Initialize logger (after logging.basicConfig)
    logger = logging.getLogger(__name__)

    try:
        # === All initialization and business logic goes here ===

        # Optional: Database initialization (only if user chose a database in Step 1)
        # db_config = {
        #     "path": config["service"]["db"]["path"],
        #     "check_same_thread": True,
        #     "timeout": config["service"]["db"]["timeout_s"],
        #     "isolation_level": None
        # }
        # sqlite_db = SqliteDB(config=db_config)
        # await sqlite_db.connect()

        logger.info('succeeded to start application')

    except Error as e:
        logger.error(f'failed to handle business error with code={e.code}, message={e.message}')
        logger.exception(e)

    except Exception as e:
        logger.error('failed to run application')
        logger.exception(e)


if __name__ == "__main__":
    asyncio.run(main())
```

---

## Feature Creation Process (CLI)

### Step 4: Initialize Feature in app/main.py (CLI)

Initialize feature modules in `app/main.py` following the dependency pattern: **DAO** → **Service**. Only create DAO if a database is configured.

---

## API Class Creation Process (CLI)

### Step 6: Initialize API Class in app/main.py (CLI)

Initialize the API class in `app/main.py`, modify at the following locations:

#### 1. Import Section (top of file)

```python
# Optional: Only import if the user chose a database in Step 1
# from app.db.SqliteDB import SqliteDB
# === Add API-related imports ===
from app.api import {Name}Api
```

#### 2. Initialization Section (inside main function, after logging initialization)

Add after `logger = logging.getLogger(__name__)`:

```python
    # === Initialize API client ===
    {name}_api = {Name}Api(config=config)

    # Optional: Database initialization (only if user chose a database in Step 1)
    # sqlite_db = SqliteDB(config=db_config)
    # await sqlite_db.connect()

    logger.info('application started')
```

#### 3: Use in Service (optional)

If the API client needs to be used by a feature's service, inject it during feature initialization:

```python
    # === Initialize feature and inject API client ===
    my_service = MyService(config=config)
    my_service.set_{name}_api(api={name}_api)  # Inject API client
```
