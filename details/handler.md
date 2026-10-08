## Handler Layer

### Module Location

`app/feature/user/handler.py`

### Handler Layer Responsibilities

1. **HTTP Request/Response Handling**: Parse request parameters, return HTTP responses
2. **Call Service Layer**: Delegate all business logic to the Service layer
3. **No Business Error Handling**: Business errors are handled uniformly by the error middleware
4. **Parameter Parsing**: Extract path parameters, query parameters, and request body from requests

For JSON object request bodies, import and use the shared `read_json_object` helper from `app.common`. It rejects malformed JSON and top-level values that are not objects with `CommonErrc.INVALID_JSON`; do not duplicate JSON parsing and shape validation in individual handlers.

### Rules

#### Parameter Naming Consistency with Service

Handler methods' local variable names should **exactly match** Service layer method parameter names, ensuring code traceability and maintainability.

For example:
- Service method `find_by_id(self, id: int)` uses parameter name `id`
- Then Handler should also use `id` when extracting from request, not `user_id`

```python
# ✅ Correct: Consistent with Service layer parameter naming (Service's find_by_id takes id)
async def find_by_id(self, request: web.Request):
    raw_id = request.match_info["id"]
    try:
        id = int(raw_id)
    except Exception as e:
        message = f'failed to parse id with raw_id={raw_id}'
        self._logger.error(message)
        raise Error(CommonErrc.INVALID_ID.value, message) from e
    user = await self._user_service.find_by_id(id=id)

# ❌ Wrong: Variable name inconsistent with Service parameter
async def find_by_id(self, request: web.Request):
    raw_user_id = request.match_info["id"]
    try:
        user_id = int(raw_user_id)
    except Exception as e:
        message = f'failed to parse id with raw_user_id={raw_user_id}'
        self._logger.error(message)
        raise Error(CommonErrc.INVALID_ID.value, message) from e
    user = await self._user_service.find_by_id(id=user_id)  # user_id does not match Service's id parameter
```

#### Service Null Check

Before executing any Service-related operations, you must check if the Service has been initialized. If the Service is not found, log the error and raise a business exception with an error code.

```python
if self._user_service is None:
    message = f'missing user_service with <relevant_params>'
    self._logger.error(message)
    raise Error(CommonErrc.MISSING_SERVICE.value, message)
```

**Notes**:
- Consistent with the DAO Null Check pattern in the Service layer
- Log format uses the `missing <component> with <params>` pattern
- Must use the common error code (e.g., `CommonErrc.MISSING_SERVICE`)

#### Method Ordering

Handler class methods should follow this order:

1. `__init__` and dependency setter methods (`set_*`)
2. **register_routes**: Route registration method, immediately after the setters
3. **insert / insert_***: Insert and related methods
4. **update / update_***: Update and related methods (e.g., update_by_id)
5. **delete / delete_***: Delete and related methods (e.g., delete_by_id)
6. **find / find_***: Find and related methods (e.g., find_by_id, find)
7. Private helper methods, after all public methods

#### Route Registration Ordering

Keep route registration order consistent with the request-operation order: insert → update → delete → find. The `register_routes` method belongs immediately after the dependency setters; it does not go at the end of the class.

#### Response Format Consistency

All Handler methods should return responses using the unified `SuccessResponse` format, with timestamp using the default value:

```python
# ✅ Correct: No data to return
        return web.json_response(SuccessResponse().to_dict())

# ✅ Correct: Return data
        return web.json_response(SuccessResponse(data={"id": id}).to_dict(), status=201)

# ❌ Wrong: Manually generate timestamp (SuccessResponse already has default)
timestamp = datetime.utcnow().isoformat() + 'Z'
return web.json_response(
    SuccessResponse(
        code='',
        data=<response_data>,
        timestamp=timestamp  # Not needed
    ).to_dict()
)
```

| Operation Type | HTTP Status Code | data Content |
|---------------|-----------------|--------------|
| insert | 201 | Created resource object |
| find / find_by_id | 200 | Resource object or list |
| update_by_id | 200 | Empty dict `{}` |
| delete_by_id | 200 | Empty dict `{}` |

### Dependency Injection Pattern

Handler uses **Setter Injection** pattern for dependency injection:

1. **Constructor injection for required dependencies**:
   - `__init__(config)` - Inject configuration dictionary

2. **Setter method injection for optional dependencies**:
   - `set_user_service(user_service)` - Inject user service


3. **Lazy initialization**:
   - Handler can be created first, dependencies injected later
   - Provides more flexible object lifecycle management

### Handler Initialization Example

```python
from app.feature.user.handler import UserHandler

# Initialize in main.py
user_handler = UserHandler(config=config)
user_handler.set_user_service(user_service=user_service)
user_handler.register_routes(app)
```

> **Note**: The `id` type in this template uses `int` as an example. In practice, the `id` type depends on the business requirements and database design (e.g., `int`, `str`, etc.).

### Complete UserHandler Template

> This template works unchanged on both SQLite and DolphinDB, because the Handler only depends on the Service layer, which itself stays DB-agnostic. Add Handler methods that mirror the Service method signatures; see `service.md`.

```python
import sys
import logging
from typing import Any

from aiohttp import web

from app.common import (
    Errc as CommonErrc,
    Error,
    SuccessResponse,
    read_json_object,
    to_orderby,
    to_page,
    to_page_size,
)
from app.feature.user.common import FieldType
from app.feature.user.service import UserService

class UserHandler:
    """User Handler"""

    _logger = logging.getLogger(__name__)

    def __init__(self, config: dict[str, Any]) -> None:
        """Initialize

        Args:
            config: Configuration dictionary
        """
        self._config = config
        self._user_service: UserService | None = None

    def set_user_service(self, user_service: UserService) -> None:
        """Set user service

        Args:
            user_service: User service
        """
        self._user_service = user_service

    def register_routes(self, app: web.Application) -> None:
        """Register routes

        Args:
            app: aiohttp application
        """
        # Order: insert → update → delete → find
        app.router.add_post("/users", self.insert)
        app.router.add_put("/users/{id}", self.update_by_id)
        app.router.add_delete("/users/{id}", self.delete_by_id)
        app.router.add_get("/users/{id}", self.find_by_id)
        app.router.add_get("/users", self.find)
        self._logger.info(f'user routes registered')

    async def insert(self, request: web.Request) -> web.Response:
        """Insert user (POST /users)

        Args:
            request: HTTP request

        Returns:
            HTTP response
        """
        payload = await read_json_object(request=request)

        if self._user_service is None:
            message = f'missing user_service with payload={payload}'
            self._logger.error(message)
            raise Error(CommonErrc.MISSING_SERVICE.value, message)

        # Call service layer
        id = await self._user_service.insert(payload=payload)

        # Return success response
        return web.json_response(SuccessResponse(data={"id": id}).to_dict(), status=201)

    async def update_by_id(self, request: web.Request) -> web.Response:
        """Update user by ID (PUT /users/{id})

        Args:
            request: HTTP request

        Returns:
            HTTP response
        """
        # Parse path parameter
        raw_id = request.match_info["id"]
        try:
            id = int(raw_id)
        except Exception as e:
            message = f'failed to parse id with raw_id={raw_id}'
            self._logger.error(message)
            raise Error(CommonErrc.INVALID_ID.value, message) from e

        payload = await read_json_object(request=request)

        if self._user_service is None:
            message = f'missing user_service with id={id}, payload={payload}'
            self._logger.error(message)
            raise Error(CommonErrc.MISSING_SERVICE.value, message)

        await self._user_service.update_by_id(id=id, payload=payload)

        # Return success response
        return web.json_response(SuccessResponse().to_dict())

    async def delete_by_id(self, request: web.Request) -> web.Response:
        """Delete user by ID (DELETE /users/{id})

        Args:
            request: HTTP request

        Returns:
            HTTP response
        """
        # Parse path parameter
        raw_id = request.match_info["id"]
        try:
            id = int(raw_id)
        except Exception as e:
            message = f'failed to parse id with raw_id={raw_id}'
            self._logger.error(message)
            raise Error(CommonErrc.INVALID_ID.value, message) from e

        # Service null check
        if self._user_service is None:
            message = f'missing user_service with id={id}'
            self._logger.error(message)
            raise Error(CommonErrc.MISSING_SERVICE.value, message)

        # Call service layer
        await self._user_service.delete_by_id(id=id)

        # Return success response
        return web.json_response(SuccessResponse().to_dict())

    async def find_by_id(self, request: web.Request) -> web.Response:
        """Find user by ID (GET /users/{id})

        Query Params:
            field_type: simple (default) or full (lowercase)

        Args:
            request: HTTP request

        Returns:
            HTTP response
        """
        # Parse path parameter
        raw_id = request.match_info["id"]
        try:
            id = int(raw_id)
        except Exception as e:
            message = f'failed to parse id with raw_id={raw_id}'
            self._logger.error(message)
            raise Error(CommonErrc.INVALID_ID.value, message) from e

        field_type = self._required_field_type(request=request)

        # Service null check
        if self._user_service is None:
            message = f'missing user_service with id={id}, field_type={field_type}'
            self._logger.error(message)
            raise Error(CommonErrc.MISSING_SERVICE.value, message)

        # Call service layer
        user_field = await self._user_service.find_by_id(id=id, field_type=field_type)

        # Return success response
        return web.json_response(
            SuccessResponse(data=user_field).to_dict()
        )

    async def find(self, request: web.Request) -> web.Response:
        """Find user list (GET /users)

        Query Params:
            field_type: simple (default) or full (lowercase)
            role_id: Filter by role ID (optional)
            username: Filter by username (optional)
            orderby: Order spec, comma-separated field:direction pairs;
                direction must be 'asc' or 'desc' (e.g., id:desc,username:asc).
                Defaults to id:desc when omitted.
            page: Page number (starting from 1, default 1)
            page_size: Number of items per page (default unlimited)

        Args:
            request: HTTP request

        Returns:
            HTTP response
        """
        field_type = self._required_field_type(request=request)

        # Pass all query parameters to the service layer
        payload: dict[str, str] = dict(request.query)

        # Parse orderby (e.g. ?orderby=id:desc,username:asc)
        raw_orderby = request.query.get("orderby")
        orderby = to_orderby(raw_orderby=raw_orderby)

        # Parse pagination
        page = to_page(raw_page=request.query.get("page"))
        page_size = to_page_size(raw_page_size=request.query.get("page_size"))

        page = 1 if page is None else page
        page_size = sys.maxsize if page_size is None else page_size

        # Service null check
        if self._user_service is None:
            message = f'missing user_service with payload={payload}, orderby={orderby}'
            self._logger.error(message)
            raise Error(CommonErrc.MISSING_SERVICE.value, message)

        # Call service layer
        items, pagination = await self._user_service.find(
            payload=payload,
            orderby=orderby,
            field_type=field_type,
            page=page,
            page_size=page_size
        )

        # Return success response
        return web.json_response(
            SuccessResponse(data={"items": items, "pagination": pagination}).to_dict()
        )

    def _required_field_type(self, request: web.Request) -> FieldType:
        """Parse the requested field type, defaulting to SIMPLE."""
        raw_field_type = request.query.get("field_type", 'simple').lower()
        try:
            return FieldType(raw_field_type)
        except Exception as e:
            message = f'failed to parse field_type with raw_field_type={raw_field_type}'
            self._logger.error(message)
            raise Error(CommonErrc.INVALID_FIELD_TYPE.value, message) from e

```
