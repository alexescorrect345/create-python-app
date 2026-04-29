---
name: create-python-app
description: Create a Python application (web service or CLI program)
---

## Triggers

### When to Use This Skill

Use this SKILL when ANY of the following conditions is met:

1. **User requests to create a new Python application**
   - Determine project type: **web service** or **CLI program**
   - This document covers shared setup; type-specific details see `cli.md` (CLI) or `web.md` (Web)

2. **User requests to create a new feature**
   - Shared process in this document; type-specific steps see `cli.md` or `web.md`

3. **User requests to create a new API class**
   - Shared process in this document; type-specific initialization see `cli.md` or `web.md`

4. **User requests to generate Web API documentation**
   - **Only applicable for web service projects** (not for CLI programs)
   - Extract route definitions from existing `handler.py` files
   - Generate `doc/WEB_API.md` following the format in `details/WEB_API.md`

5. **User explicitly forces the use of this skill**

---

## Expected Output

When creating a project or feature, provide:

1. **Project Structure**: Complete directory tree with all necessary files
2. **Generated Files**: All template files from `details/` directory
3. **Next Steps**: Clear instructions on how to run and use the project

---

## Tool Conventions

- **Dependency Management**: Poetry
- **Program Start/Stop**: Poetry (poetry run python app/main.py)

---

## Project Creation Process

### Step 1: Confirm Project Type and Database Selection

> **⚠️ CLI vs Web Difference**
>
> **Project Type**:
> - **CLI**: Command-line program, no HTTP service
> - **Web**: Web service, provides HTTP API
>
> **Database Selection** (same for both CLI and Web):
> - Database is optional (supports None / SQLite / DolphinDB). If None is chosen, skip all database-related steps

Ask user to determine project type and database type:

> "Does this project need a database?"
> - **None**: No database needed
> - **SQLite**: Lightweight embedded database, suitable for small projects and rapid prototyping
> - **DolphinDB**: High-performance time-series database, suitable for big data analytics and time-series data processing

If the user chooses **None**, skip all database-related steps (Step 8, Step 9, database-related configuration, etc.).

### Step 2: Create Directory Structure

> **⚠️ CLI vs Web Difference**
> - **Web** additionally includes: `app/middleware.py`, `doc/WEB_API.md`, `handler.py` (feature level)
> - **CLI** directory structure is more minimal
>
> For detailed directory structure, see `cli.md` or `web.md`

### Step 3: Create poetry.toml and pyproject.toml

Create `poetry.toml` and `pyproject.toml` with the following content. If `poetry.toml` already exists in the project, verify that its contents are consistent with the example below.

**poetry.toml:**
```toml
[virtualenvs]
in-project = true
create = true
```

**pyproject.toml:**
If `pyproject.toml` already exists in the project, ensure the following configuration items are present (add any missing ones):

```toml
[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"

[project]
name = "my-app"
version = "0.1.0"
description = "A Python CLI program"    # CLI
# description = "A Python web service"  # Web
readme = "README.md"
requires-python = ">=3.9"
authors = [{name = "Developer", email = "dev@example.com"}]

dependencies = [
    "aiohttp>=3.9.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0",
    "ruff",
]

[tool.ruff]
line-length = 88
```

> **⚠️ CLI vs Web Difference**
> - The `description` field differs: CLI uses `"A Python CLI program"`, Web uses `"A Python web service"`

### Step 4: Create Configuration Files

Create `config/` directory and configuration files. The configuration system supports multiple environments through different config files. By default, two configuration files are provided:

- `config/config.dev.toml` — for non-production environments (development, testing)
- `config/config.toml` — for production environment

Users can also create additional config files (e.g., `config.staging.toml`, `config.local.toml`) as needed.

#### Log Configuration (Shared)

```toml
# Log configuration
[log]
main = "DEBUG"  # config.dev.toml, supports DEBUG, INFO, WARNING, ERROR, CRITICAL
# main = "INFO" # config.toml
```

> **⚠️ CLI vs Web Difference**
>
> **Web** additionally includes the `[web]` config section (host / port / body_limit / cors).
> **CLI** has no `[web]` config section.
>
> **Database Configuration** (same for both CLI and Web):
> - Only added when the user chose a database in Step 1 (not added when None is chosen)
>
> For detailed config file content, see `cli.md` or `web.md`

### Step 5: Create app/common.py

> **⚠️ CLI vs Web Difference**
> - `Errc` enum error codes are shared between CLI and Web (defined below)

#### Errc Enum

```python
class Errc(Enum):
    """Common error code enum"""

    # HTTP related errors
    UNKNOWN_ERROR = 'myapp::common::000'
    RESOURCE_NOT_FOUND = 'myapp::common::001'
    METHOD_NOT_ALLOWED = 'myapp::common::002'

    # Common errors
    INTERNAL_SERVER_ERROR = 'myapp::common::003'
    BAD_REQUEST = 'myapp::common::004'
    UNAUTHORIZED = 'myapp::common::005'
    FORBIDDEN = 'myapp::common::006'

    # Timeout errors
    TIMEOUT = 'myapp::common::007'

    # INVALID errors
    INVALID_PARAMETER = 'myapp::common::008'
    INVALID_JSON = 'myapp::common::009'
    INVALID_TYPE = 'myapp::common::010'

    # MISSING errors
    MISSING_PARAMETER = 'myapp::common::011'
    MISSING_CONFIG = 'myapp::common::012'
    MISSING_SERVICE = 'myapp::common::013'
    MISSING_DAO = 'myapp::common::014'
    MISSING_API = 'myapp::common::015'
    MISSING_FIELD = 'myapp::common::016'
```

#### Shared Classes

```python
from enum import Enum
from typing import Any, Dict
from datetime import datetime, timezone
from dataclasses import dataclass, field, asdict


class Error(Exception):
    """Business error exception class"""

    def __init__(self, code: str, message: str):
        """Initialize error

        Args:
            code: Error code
            message: Error message
        """
        self.code = code
        self.message = message
        super().__init__(self.message)

    def __str__(self) -> str:
        """Return error string representation"""
        return f'Error({self.code}, {self.message})'


@dataclass
class SuccessResponse:
    """Data structure for returning success response to HTTP API callers"""

    code: str = ''
    data: Any = field(default_factory=dict)
    timestamp: int = field(default_factory=lambda: int(datetime.now(timezone.utc).timestamp() * 1000))

    def to_dict(self) -> Dict[str, Any]:
        """Convert to dictionary

        Returns:
            Response dictionary
        """
        return asdict(self)

    @classmethod
    def from_dict(cls, data: Dict[str, Any]) -> 'SuccessResponse':
        """Create instance from dictionary

        Args:
            data: Dictionary containing response data

        Returns:
            SuccessResponse instance
        """
        return cls(**data)

    def __str__(self) -> str:
        """Return string representation"""
        ts = datetime.fromtimestamp(self.timestamp / 1000, tz=timezone.utc).strftime('%Y-%m-%dT%H:%M:%S.%f')[:-3]
        return f'SuccessResponse(code={self.code}, data={self.data}, timestamp={ts})'


@dataclass
class ErrorResponse:
    """Data structure for returning error response to HTTP API callers"""

    code: str
    message: str
    timestamp: int = field(default_factory=lambda: int(datetime.now(timezone.utc).timestamp() * 1000))

    def to_dict(self) -> Dict[str, Any]:
        """Convert to dictionary

        Returns:
            Response dictionary
        """
        return asdict(self)

    @classmethod
    def from_dict(cls, data: Dict[str, Any]) -> 'ErrorResponse':
        """Create instance from dictionary

        Args:
            data: Dictionary containing response data

        Returns:
            ErrorResponse instance
        """
        return cls(**data)

    def __str__(self) -> str:
        """Return string representation"""
        ts = datetime.fromtimestamp(self.timestamp / 1000, tz=timezone.utc).strftime('%Y-%m-%dT%H:%M:%S.%f')[:-3]
        return f'ErrorResponse(code={self.code}, message={self.message}, timestamp={ts})'
```

#### Pagination

```python
@dataclass
class Pagination:
    """Pagination info for API response"""

    page: int
    page_size: int
    total: int
    total_pages: int
    has_next: bool
    has_prev: bool

    def to_dict(self) -> Dict[str, Any]:
        """Convert to dictionary

        Returns:
            Pagination dictionary
        """
        return asdict(self)

    @classmethod
    def from_dict(cls, data: Dict[str, Any]) -> 'Pagination':
        """Create instance from dictionary

        Args:
            data: Dictionary containing pagination data

        Returns:
            Pagination instance
        """
        return cls(**data)

    def __str__(self) -> str:
        """Return string representation"""
        return f'Pagination(page={self.page}, page_size={self.page_size}, total={self.total}, total_pages={self.total_pages}, has_next={self.has_next}, has_prev={self.has_prev})'
```

### Step 6: Create app/api/__init__.py

Create `app/api/__init__.py` with the following content:

```python
"""API module"""

# Add API client class exports here, e.g.:
# from app.api.{Name}Api import {Name}Api

__all__ = []
```

### Step 7: Create app/api/common.py

Create `app/api/common.py` with the following content:

```python
"""API module common error codes"""

from enum import Enum
from typing import Any, Dict, Optional


class Errc(Enum):
    """API error code enum"""
    UNKNOWN_ERROR = 'myapp::api::000'
    SESSION_NOT_FOUND = 'myapp::api::001'
    API_REQUEST_FAILED = 'myapp::api::002'


def sanitize_headers(headers: Optional[Dict[str, str]]) -> Optional[Dict[str, str]]:
    """Remove sensitive fields from headers for logging"""
    if headers is None:
        return None
    return {k: v for k, v in headers.items() if k.lower() not in ('cookie', 'authorization')}
```

### Step 8: Create app/api/api.py

Create `app/api/api.py` with the following content:

```python
"""API base classes - InternalApi for SuccessResponse/ErrorResponse format APIs, ExternalApi for raw JSON APIs."""
import time
import logging
from typing import Dict, Any, Optional

import aiohttp

from app.common import Error, Errc as CommonErrc, SuccessResponse, ErrorResponse
from app.api.common import Errc as ApiErrc, sanitize_headers


class InternalApi:
    """Internal API base class

    Calls HTTP APIs created by this project's skills
    Response format is SuccessResponse / ErrorResponse
    """

    _logger = logging.getLogger(__name__)

    def __init__(self, api_config: Dict[str, Any]):
        """Initialize API client

        Args:
            api_config: API configuration dict, e.g. config["service"]["api"]["myinternalapi"]
                        Must contain 'base_url' and 'timeout_s'
        """
        self._api_config = api_config
        self._session = None

    def init(self):
        """Initialize the session, should be called after event loop is created"""
        timeout = aiohttp.ClientTimeout(total=self._api_config["timeout_s"])
        self._session = aiohttp.ClientSession(timeout=timeout)

    async def close(self):
        """Close session"""
        if self._session:
            try:
                await self._session.close()
            except Exception as e:
                message = f'failed to close session with base_url={self._api_config["base_url"]}'
                self._logger.error(message)
                self._logger.exception(e)

    async def _get(
        self,
        url: str,
        params: Optional[Dict[str, Any]] = None,
        headers: Optional[Dict[str, str]] = None,
    ) -> SuccessResponse:
        """Send GET request

        Args:
            url: Request path, e.g. '/users/1'
            params: Query parameters
            headers: Request headers

        Returns:
            SuccessResponse object

        Raises:
            Error: API request failed
        """
        if not self._session:
            message = f'failed to find session with base_url={self._api_config["base_url"]}'
            self._logger.error(message)
            raise Error(ApiErrc.SESSION_NOT_FOUND.value, message)

        base_url = self._api_config["base_url"].rstrip('/')
        full_url = f'{base_url}{url}'
        start_time = time.time()
        safe_headers = sanitize_headers(headers)

        try:
            async with self._session.get(full_url, params=params, headers=headers) as response:
                elapsed = time.time() - start_time
                if not (200 <= response.status < 300):
                    error_text = await response.text()
                    message = f'[{elapsed:.3f}s]failed to get with url={full_url}, params={params}, headers={safe_headers}, status={response.status}, error={error_text}'
                    self._logger.error(message)
                    raise Error(ApiErrc.API_REQUEST_FAILED.value, message)

                result = await response.json()

                if result["code"] != '':
                    error_response = ErrorResponse.from_dict(result)
                    message = f'[{elapsed:.3f}s]failed to get with API error: code={error_response.code}, message={error_response.message}, url={full_url}, params={params}, headers={safe_headers}'
                    self._logger.error(message)
                    raise Error(error_response.code, message)

                success_response = SuccessResponse.from_dict(result)
                self._logger.debug(f'[{elapsed:.3f}s]succeeded to get with url={full_url}, params={params}, headers={safe_headers}')
                return success_response

        except Error:
            raise

        except TimeoutError as e:
            elapsed = time.time() - start_time
            message = f'[{elapsed:.3f}s]timeout to get with url={full_url}, params={params}, headers={safe_headers}'
            self._logger.error(message)
            raise Error(CommonErrc.TIMEOUT.value, message) from e

        except Exception as e:
            elapsed = time.time() - start_time
            message = f'[{elapsed:.3f}s]failed to get with url={full_url}, params={params}, headers={safe_headers}'
            self._logger.error(message)
            raise Error(ApiErrc.API_REQUEST_FAILED.value, message) from e

    async def _post(
        self,
        url: str,
        payload: Optional[Dict[str, Any]] = None,
        headers: Optional[Dict[str, str]] = None,
    ) -> SuccessResponse:
        """Send POST request

        Args:
            url: Request path, e.g. '/users'
            payload: Request body
            headers: Request headers

        Returns:
            SuccessResponse object

        Raises:
            Error: API request failed
        """
        if not self._session:
            message = f'failed to find session with base_url={self._api_config["base_url"]}'
            self._logger.error(message)
            raise Error(ApiErrc.SESSION_NOT_FOUND.value, message)

        base_url = self._api_config["base_url"].rstrip('/')
        full_url = f'{base_url}{url}'
        start_time = time.time()
        safe_headers = sanitize_headers(headers)

        try:
            async with self._session.post(full_url, json=payload, headers=headers) as response:
                elapsed = time.time() - start_time
                if not (200 <= response.status < 300):
                    error_text = await response.text()
                    message = f'[{elapsed:.3f}s]failed to post with url={full_url}, payload={payload}, headers={safe_headers}, status={response.status}, error={error_text}'
                    self._logger.error(message)
                    raise Error(ApiErrc.API_REQUEST_FAILED.value, message)

                result = await response.json()

                if result["code"] != '':
                    error_response = ErrorResponse.from_dict(result)
                    message = f'[{elapsed:.3f}s]failed to post with API error: code={error_response.code}, message={error_response.message}, url={full_url}, payload={payload}, headers={safe_headers}'
                    self._logger.error(message)
                    raise Error(error_response.code, message)

                success_response = SuccessResponse.from_dict(result)
                self._logger.debug(f'[{elapsed:.3f}s]succeeded to post with url={full_url}, payload={payload}, headers={safe_headers}')
                return success_response

        except Error:
            raise

        except TimeoutError as e:
            elapsed = time.time() - start_time
            message = f'[{elapsed:.3f}s]timeout to post with url={full_url}, payload={payload}, headers={safe_headers}'
            self._logger.error(message)
            raise Error(CommonErrc.TIMEOUT.value, message) from e

        except Exception as e:
            elapsed = time.time() - start_time
            message = f'[{elapsed:.3f}s]failed to post with url={full_url}, payload={payload}, headers={safe_headers}'
            self._logger.error(message)
            raise Error(ApiErrc.API_REQUEST_FAILED.value, message) from e

    async def _put(
        self,
        url: str,
        payload: Optional[Dict[str, Any]] = None,
        headers: Optional[Dict[str, str]] = None,
    ) -> SuccessResponse:
        """Send PUT request

        Args:
            url: Request path, e.g. '/users/1'
            payload: Request body
            headers: Request headers

        Returns:
            SuccessResponse object

        Raises:
            Error: API request failed
        """
        if not self._session:
            message = f'failed to find session with base_url={self._api_config["base_url"]}'
            self._logger.error(message)
            raise Error(ApiErrc.SESSION_NOT_FOUND.value, message)

        base_url = self._api_config["base_url"].rstrip('/')
        full_url = f'{base_url}{url}'
        start_time = time.time()
        safe_headers = sanitize_headers(headers)

        try:
            async with self._session.put(full_url, json=payload, headers=headers) as response:
                elapsed = time.time() - start_time
                if not (200 <= response.status < 300):
                    error_text = await response.text()
                    message = f'[{elapsed:.3f}s]failed to put with url={full_url}, payload={payload}, headers={safe_headers}, status={response.status}, error={error_text}'
                    self._logger.error(message)
                    raise Error(ApiErrc.API_REQUEST_FAILED.value, message)

                result = await response.json()

                if result["code"] != '':
                    error_response = ErrorResponse.from_dict(result)
                    message = f'[{elapsed:.3f}s]failed to put with API error: code={error_response.code}, message={error_response.message}, url={full_url}, payload={payload}, headers={safe_headers}'
                    self._logger.error(message)
                    raise Error(error_response.code, message)

                success_response = SuccessResponse.from_dict(result)
                self._logger.debug(f'[{elapsed:.3f}s]succeeded to put with url={full_url}, payload={payload}, headers={safe_headers}')
                return success_response

        except Error:
            raise

        except TimeoutError as e:
            elapsed = time.time() - start_time
            message = f'[{elapsed:.3f}s]timeout to put with url={full_url}, payload={payload}, headers={safe_headers}'
            self._logger.error(message)
            raise Error(CommonErrc.TIMEOUT.value, message) from e

        except Exception as e:
            elapsed = time.time() - start_time
            message = f'[{elapsed:.3f}s]failed to put with url={full_url}, payload={payload}, headers={safe_headers}'
            self._logger.error(message)
            raise Error(ApiErrc.API_REQUEST_FAILED.value, message) from e

    async def _delete(
        self,
        url: str,
        headers: Optional[Dict[str, str]] = None,
    ) -> None:
        """Send DELETE request

        Args:
            url: Request path, e.g. '/users/1'
            headers: Request headers

        Raises:
            Error: API request failed
        """
        if not self._session:
            message = f'failed to find session with base_url={self._api_config["base_url"]}'
            self._logger.error(message)
            raise Error(ApiErrc.SESSION_NOT_FOUND.value, message)

        base_url = self._api_config["base_url"].rstrip('/')
        full_url = f'{base_url}{url}'
        start_time = time.time()
        safe_headers = sanitize_headers(headers)

        try:
            async with self._session.delete(full_url, headers=headers) as response:
                elapsed = time.time() - start_time
                if not (200 <= response.status < 300):
                    error_text = await response.text()
                    message = f'[{elapsed:.3f}s]failed to delete with url={full_url}, headers={safe_headers}, status={response.status}, error={error_text}'
                    self._logger.error(message)
                    raise Error(ApiErrc.API_REQUEST_FAILED.value, message)

                self._logger.debug(f'[{elapsed:.3f}s]succeeded to delete with url={full_url}, headers={safe_headers}')

        except Error:
            raise

        except TimeoutError as e:
            elapsed = time.time() - start_time
            message = f'[{elapsed:.3f}s]timeout to delete with url={full_url}, headers={safe_headers}'
            self._logger.error(message)
            raise Error(CommonErrc.TIMEOUT.value, message) from e

        except Exception as e:
            elapsed = time.time() - start_time
            message = f'[{elapsed:.3f}s]failed to delete with url={full_url}, headers={safe_headers}'
            self._logger.error(message)
            raise Error(ApiErrc.API_REQUEST_FAILED.value, message) from e


class ExternalApi:
    """External API base class

    Calls third-party APIs
    Returns raw JSON, caller parses response structure on their own
    """

    _logger = logging.getLogger(__name__)

    def __init__(self, api_config: Dict[str, Any]):
        """Initialize API client

        Args:
            api_config: API configuration dict, e.g. config["service"]["api"]["myexternalapi"]
                        Must contain 'base_url' and 'timeout_s'
        """
        self._api_config = api_config
        self._session = None

    def init(self):
        """Initialize the session, should be called after event loop is created"""
        timeout = aiohttp.ClientTimeout(total=self._api_config["timeout_s"])
        self._session = aiohttp.ClientSession(timeout=timeout)

    async def close(self):
        """Close session"""
        if self._session:
            try:
                await self._session.close()
            except Exception as e:
                message = f'failed to close session with base_url={self._api_config["base_url"]}'
                self._logger.error(message)
                self._logger.exception(e)

    async def _get(
        self,
        url: str,
        params: Optional[Dict[str, Any]] = None,
        headers: Optional[Dict[str, str]] = None,
    ) -> Any:
        """Send GET request

        Args:
            url: Request path, e.g. '/data'
            params: Query parameters
            headers: Request headers

        Returns:
            Raw JSON response

        Raises:
            Error: API request failed
        """
        if not self._session:
            message = f'failed to find session with base_url={self._api_config["base_url"]}'
            self._logger.error(message)
            raise Error(ApiErrc.SESSION_NOT_FOUND.value, message)

        base_url = self._api_config["base_url"].rstrip('/')
        full_url = f'{base_url}{url}'
        start_time = time.time()
        safe_headers = sanitize_headers(headers)

        try:
            async with self._session.get(full_url, params=params, headers=headers) as response:
                elapsed = time.time() - start_time
                if not (200 <= response.status < 300):
                    error_text = await response.text()
                    message = f'[{elapsed:.3f}s]failed to get with url={full_url}, params={params}, headers={safe_headers}, status={response.status}, error={error_text}'
                    self._logger.error(message)
                    raise Error(ApiErrc.API_REQUEST_FAILED.value, message)

                result = await response.json()
                self._logger.debug(f'[{elapsed:.3f}s]succeeded to get with url={full_url}, params={params}, headers={safe_headers}')
                return result

        except Error:
            raise

        except TimeoutError as e:
            elapsed = time.time() - start_time
            message = f'[{elapsed:.3f}s]timeout to get with url={full_url}, params={params}, headers={safe_headers}'
            self._logger.error(message)
            raise Error(CommonErrc.TIMEOUT.value, message) from e

        except Exception as e:
            elapsed = time.time() - start_time
            message = f'[{elapsed:.3f}s]failed to get with url={full_url}, params={params}, headers={safe_headers}'
            self._logger.error(message)
            raise Error(ApiErrc.API_REQUEST_FAILED.value, message) from e

    async def _post(
        self,
        url: str,
        payload: Optional[Dict[str, Any]] = None,
        headers: Optional[Dict[str, str]] = None,
    ) -> Any:
        """Send POST request

        Args:
            url: Request path, e.g. '/data'
            payload: Request body
            headers: Request headers

        Returns:
            Raw JSON response

        Raises:
            Error: API request failed
        """
        if not self._session:
            message = f'failed to find session with base_url={self._api_config["base_url"]}'
            self._logger.error(message)
            raise Error(ApiErrc.SESSION_NOT_FOUND.value, message)

        base_url = self._api_config["base_url"].rstrip('/')
        full_url = f'{base_url}{url}'
        start_time = time.time()
        safe_headers = sanitize_headers(headers)

        try:
            async with self._session.post(full_url, json=payload, headers=headers) as response:
                elapsed = time.time() - start_time
                if not (200 <= response.status < 300):
                    error_text = await response.text()
                    message = f'[{elapsed:.3f}s]failed to post with url={full_url}, payload={payload}, headers={safe_headers}, status={response.status}, error={error_text}'
                    self._logger.error(message)
                    raise Error(ApiErrc.API_REQUEST_FAILED.value, message)

                result = await response.json()
                self._logger.debug(f'[{elapsed:.3f}s]succeeded to post with url={full_url}, payload={payload}, headers={safe_headers}')
                return result

        except Error:
            raise

        except TimeoutError as e:
            elapsed = time.time() - start_time
            message = f'[{elapsed:.3f}s]timeout to post with url={full_url}, payload={payload}, headers={safe_headers}'
            self._logger.error(message)
            raise Error(CommonErrc.TIMEOUT.value, message) from e

        except Exception as e:
            elapsed = time.time() - start_time
            message = f'[{elapsed:.3f}s]failed to post with url={full_url}, payload={payload}, headers={safe_headers}'
            self._logger.error(message)
            raise Error(ApiErrc.API_REQUEST_FAILED.value, message) from e

    async def _put(
        self,
        url: str,
        payload: Optional[Dict[str, Any]] = None,
        headers: Optional[Dict[str, str]] = None,
    ) -> Any:
        """Send PUT request

        Args:
            url: Request path, e.g. '/data/1'
            payload: Request body
            headers: Request headers

        Returns:
            Raw JSON response

        Raises:
            Error: API request failed
        """
        if not self._session:
            message = f'failed to find session with base_url={self._api_config["base_url"]}'
            self._logger.error(message)
            raise Error(ApiErrc.SESSION_NOT_FOUND.value, message)

        base_url = self._api_config["base_url"].rstrip('/')
        full_url = f'{base_url}{url}'
        start_time = time.time()
        safe_headers = sanitize_headers(headers)

        try:
            async with self._session.put(full_url, json=payload, headers=headers) as response:
                elapsed = time.time() - start_time
                if not (200 <= response.status < 300):
                    error_text = await response.text()
                    message = f'[{elapsed:.3f}s]failed to put with url={full_url}, payload={payload}, headers={safe_headers}, status={response.status}, error={error_text}'
                    self._logger.error(message)
                    raise Error(ApiErrc.API_REQUEST_FAILED.value, message)

                result = await response.json()
                self._logger.debug(f'[{elapsed:.3f}s]succeeded to put with url={full_url}, payload={payload}, headers={safe_headers}')
                return result

        except Error:
            raise

        except TimeoutError as e:
            elapsed = time.time() - start_time
            message = f'[{elapsed:.3f}s]timeout to put with url={full_url}, payload={payload}, headers={safe_headers}'
            self._logger.error(message)
            raise Error(CommonErrc.TIMEOUT.value, message) from e

        except Exception as e:
            elapsed = time.time() - start_time
            message = f'[{elapsed:.3f}s]failed to put with url={full_url}, payload={payload}, headers={safe_headers}'
            self._logger.error(message)
            raise Error(ApiErrc.API_REQUEST_FAILED.value, message) from e

    async def _delete(
        self,
        url: str,
        headers: Optional[Dict[str, str]] = None,
    ) -> None:
        """Send DELETE request

        Args:
            url: Request path, e.g. '/data/1'
            headers: Request headers

        Raises:
            Error: API request failed
        """
        if not self._session:
            message = f'failed to find session with base_url={self._api_config["base_url"]}'
            self._logger.error(message)
            raise Error(ApiErrc.SESSION_NOT_FOUND.value, message)

        base_url = self._api_config["base_url"].rstrip('/')
        full_url = f'{base_url}{url}'
        start_time = time.time()
        safe_headers = sanitize_headers(headers)

        try:
            async with self._session.delete(full_url, headers=headers) as response:
                elapsed = time.time() - start_time
                if not (200 <= response.status < 300):
                    error_text = await response.text()
                    message = f'[{elapsed:.3f}s]failed to delete with url={full_url}, headers={safe_headers}, status={response.status}, error={error_text}'
                    self._logger.error(message)
                    raise Error(ApiErrc.API_REQUEST_FAILED.value, message)

                self._logger.debug(f'[{elapsed:.3f}s]succeeded to delete with url={full_url}, headers={safe_headers}')

        except Error:
            raise

        except TimeoutError as e:
            elapsed = time.time() - start_time
            message = f'[{elapsed:.3f}s]timeout to delete with url={full_url}, headers={safe_headers}'
            self._logger.error(message)
            raise Error(CommonErrc.TIMEOUT.value, message) from e

        except Exception as e:
            elapsed = time.time() - start_time
            message = f'[{elapsed:.3f}s]failed to delete with url={full_url}, headers={safe_headers}'
            self._logger.error(message)
            raise Error(ApiErrc.API_REQUEST_FAILED.value, message) from e
```

### Step 9: Create Database Layer

This step is optional for both CLI and Web. Only execute when the user chose a database in Step 1. Skip if None was chosen.

Create the database layer files in `app/db/`:

#### 9.1 Create `app/db/__init__.py`

```python
"""Database module"""

from app.db import DB

__all__ = ['DB']
```

#### 9.2 Create `app/db/DB.py`

```python
from abc import ABC, abstractmethod
from typing import Any, Dict, List, Optional, Tuple


class DB(ABC):
    """Database abstract base class"""

    def __init__(self, config: Dict[str, Any]) -> None:
        """Initialize database

        Args:
            config: Configuration dictionary
        """
        self._config = config
        self._connection = None

    def is_connected(self) -> bool:
        """Whether database is connected"""
        return self._connection is not None

    @abstractmethod
    async def connect(self) -> None:
        """Connect to database"""
        pass

    @abstractmethod
    async def close(self) -> None:
        """Close database connection"""
        pass

    @abstractmethod
    async def exec(
        self,
        script: str,
        params: Optional[Tuple[Any, ...]] = None
    ) -> Optional[List[Tuple[Any, ...]]]:
        """Execute SQL script

        Args:
            script: SQL script
            params: Parameter tuple

        Returns:
            Query result list or None
        """
        pass

    @abstractmethod
    async def batch_exec(
        self,
        scripts: List[str],
        params_list: List[Optional[Tuple[Any, ...]]] = []
    ) -> List[List[Tuple[Any, ...]]]:
        """Batch execute SQL scripts

        Args:
            scripts: SQL script list
            params_list: Parameter tuple list, default empty list

        Returns:
            Query result list (only query scripts), or empty list if no query scripts
        """
        pass
```

#### 9.3 Create `app/db/common.py`

```python
"""Database module common error codes"""

from enum import Enum


class Errc(Enum):
    """Database error code enum"""

    # Common errors
    UNKNOWN_ERROR = 'myapp::db::000'

    # Configuration errors
    MISSING_DB_PATH = 'myapp::db::001'

    # Connection errors
    FAILED_TO_CONNECT = 'myapp::db::002'
    FAILED_TO_DISCONNECT = 'myapp::db::003'
    NOT_CONNECTED = 'myapp::db::004'

    # Filesystem errors
    FAILED_TO_MK_DB_DIR = 'myapp::db::005'

    # Execution errors
    MISSING_SCRIPT = 'myapp::db::006'
    FAILED_TO_EXEC = 'myapp::db::007'
```

#### 9.4 Create `app/db/{Name}DB.py` (Optional)

Only execute this step if the user selected a database in Step 1 (user chose SQLite/DolphinDB).

Based on the database type selected by the user, create the corresponding database implementation class (e.g., `SqliteDB.py`) following the steps in `details/db/create-sqlite-database.md`.

### Step 10: Create Background Task Base Class

This step is optional for both CLI and Web. Only execute when the user chose a database in Step 1.

Create `app/task.py` with the following content:

```python
import logging
from abc import ABC, abstractmethod
from typing import Any, Dict

from app.common import Error


class Task(ABC):
    """Background task base class"""

    _logger = logging.getLogger(__name__)

    def __init__(self, config: Dict[str, Any]) -> None:
        """Initialize

        Args:
            config: Configuration dictionary
        """
        self._config = config
        self._running = False

    @abstractmethod
    async def _run_once(self) -> None:
        """Execute task once"""
        pass

    async def run_in_loop(self) -> None:
        """Run task in loop"""
        self._running = True
        while self._running:
            try:
                await self._run_once()
            except Error as e:
                self._logger.error(f'failed to run task loop with code={e.code}, message={e.message}')
                self._logger.exception(e)
                self._running = False
            except Exception as e:
                self._logger.error('failed to run task loop')
                self._logger.exception(e)
                self._running = False

    async def close(self) -> None:
        """Close task"""
        self._running = False
        self._logger.info('task closed')

    def is_running(self) -> bool:
        """Whether task is running"""
        return self._running
```

### Step 11: Create app/main.py

> **⚠️ CLI vs Web Difference — Completely Different**
> - **CLI**: Uses `asyncio.run(main())` to start a pure async program
> - **Web**: Uses `web.run_app(app)` to start an aiohttp web service
>
> For detailed code, see `cli.md` or `web.md`

---

## Feature Creation Process

### Step 1: Create Directory Structure

> **⚠️ CLI vs Web Difference**
>
> **CLI** — `app/feature/{name}/` contains:
> - `__init__.py`
> - `field.py` - Data transfer object
> - `dao.py` - Data access object (optional, only when database is needed)
> - `service.py` - Business logic layer
> - `task.py` - Background task (optional)
>
> **Web** — `app/feature/{name}/` contains:
> - `__init__.py`
> - `common.py` - Error codes (Errc)
> - `field.py` - Data transfer object
> - `dao.py` - Data access object (optional, only when database is needed)
> - `service.py` - Business logic layer
> - `handler.py` - HTTP handler
> - `task.py` - Background task (optional)

### Step 2: Generate Template Files

Generate all template files from `details/` directory:

> **CLI**:
> - `details/field.md` → `app/feature/{name}/field.py`
> - `details/dao.md` → `app/feature/{name}/dao.py` (optional, only when database is needed)
> - `details/service.md` → `app/feature/{name}/service.py`
> - `details/task.md` → `app/feature/{name}/task.py` (optional)
>
> **Web**:
> - `details/field.md` → `app/feature/{name}/field.py`
> - `details/dao.md` → `app/feature/{name}/dao.py` (optional, only when database is needed)
> - `details/service.md` → `app/feature/{name}/service.py`
> - `details/handler.md` → `app/feature/{name}/handler.py`
> - `details/task.md` → `app/feature/{name}/task.py` (optional)

### Step 3: Configure Package Exports (__init__.py)

Create `app/feature/{name}/__init__.py` to export all classes from this feature module. This allows importing with the shorter path `from app.feature.{name} import {Class}` instead of `from app.feature.{name}.{module} import {Class}`.

> **⚠️ CLI vs Web Difference**
>
> **CLI**:
> ```python
> # app/feature/{name}/__init__.py
> """Feature module - {name}"""
>
> from app.feature.{name}.field import {Name}Field
> from app.feature.{name}.service import {Name}Service
> # Optional (only when database is needed): from app.feature.{name}.dao import {Name}Dao
> # Optional: from app.feature.{name}.task import {Name}Task
>
> __all__ = [
>     "{Name}Field",
>     "{Name}Service",
>     # Optional: "{Name}Dao",
>     # Optional: "{Name}Task",
> ]
> ```
>
> **Web**:
> ```python
> # app/feature/{name}/__init__.py
> """Feature module - {name}"""
>
> from app.feature.{name}.field import {Name}Field
> from app.feature.{name}.service import {Name}Service
> from app.feature.{name}.handler import {Name}Handler
> # Optional (only when database is needed): from app.feature.{name}.dao import {Name}Dao
> # Optional: from app.feature.{name}.task import {Name}Task
>
> __all__ = [
>     "{Name}Field",
>     "{Name}Service",
>     "{Name}Handler",
>     # Optional: "{Name}Dao",
>     # Optional: "{Name}Task",
> ]
> ```

**Usage Example:**

> **CLI**:
> ```python
> # ✅ Correct: Use shorter import path
> from app.feature.user import UserField, UserService
>
> # ❌ Incorrect: Do not use full module path
> from app.feature.user.field import UserField
> from app.feature.user.service import UserService
> ```
>
> **Web**:
> ```python
> # ✅ Correct: Use shorter import path
> from app.feature.user import UserField, UserDao, UserService, UserHandler
>
> # ❌ Incorrect: Do not use full module path
> from app.feature.user.field import UserField
> from app.feature.user.dao import UserDao
> from app.feature.user.service import UserService
> from app.feature.user.handler import UserHandler
> ```

### Step 4: Initialize Feature in app/main.py

> **⚠️ CLI vs Web Difference**
>
> **CLI** — Initialization order: **DAO** → **Service**. Only create DAO when a database is configured.
>
> **Web** — Initialization order: **DAO** (optional, only when database is needed) → **Service** → **Handler** → **Register Routes**.
> Also needs to initialize data tables in `on_startup` and store instances in the `app` dictionary.
>
> For detailed code, see `cli.md` or `web.md`

---

## API Class Creation Process

### Step 1: Create Directory Structure

Create `app/api/` directory with:
- `__init__.py` - Package initialization
- `{Name}Api.py` - API client class for accessing external interfaces

### Step 2: Generate Template Files

Generate `{Name}Api.py` from `details/api.md` template. Select the appropriate template based on API type:

- **InternalApi**: Calls HTTP APIs created by this project's skills, response format is `SuccessResponse` / `ErrorResponse`
- **ExternalApi**: Calls third-party APIs, returns raw JSON for the caller to parse

### Step 3: Implement API Client Class

Create `{Name}Api.py` following the templates defined in `details/api.md`.

### Step 4: Configure Package Exports (__init__.py)

Update `app/api/__init__.py` to export the newly created API client class. This allows importing with the shorter path `from app.api import {Name}Api` instead of `from app.api.{Name}Api import {Name}Api`.

```python
# app/api/__init__.py
"""API module"""

# Add API client class exports here, e.g.:
# from app.api.{Name}Api import {Name}Api

__all__ = []
```

**Usage Example:**

```python
# ✅ Correct: Use shorter import path
from app.api import {Name}Api

# ❌ Incorrect: Do not use full module path
from app.api.{Name}Api import {Name}Api
```

### Step 5: Add Configuration

Add API configuration to `config/config.dev.toml` and `config/config.toml`:

```toml
[service.api.{name}]
base_url = "http://localhost:8000"
timeout_s = 5
```

Where `{name}` comes from the `{Name}Api.py` filename.

### Step 6: Initialize API Class in app/main.py

> **⚠️ CLI vs Web Difference**
>
> **CLI** — Simple initialization: create API instance in `main()` function and inject into Service.
>
> **Web** — Requires closing API session in `on_cleanup` lifecycle hook, and storing instance in the `app` dictionary.
>
> For detailed code, see `cli.md` or `web.md`
