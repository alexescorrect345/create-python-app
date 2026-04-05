## Handler Layer

### Module Location

`app/feature/user/handler.py`

### Handler Layer Responsibilities

1. **HTTP Request/Response Handling**: Parse request parameters, return HTTP responses
2. **Call Service Layer**: Delegate all business logic to the Service layer
3. **No Business Error Handling**: Business errors are handled uniformly by the error middleware
4. **Parameter Parsing**: Extract path parameters, query parameters, and request body from requests

### Rules

#### Parameter Naming Consistency with Service

Handler methods' local variable names should **exactly match** Service layer method parameter names, ensuring code traceability and maintainability.

For example:
- Service method `find_by_id(self, id: int)` uses parameter name `id`
- Then Handler should also use `id` when extracting from request, not `user_id`

```python
# ✅ Correct: Consistent with Service layer parameter naming (Service's find_by_id takes id)
async def find_by_id(self, request: web.Request):
    id = int(request.match_info["id"])
    user = await self._user_service.find_by_id(id=id)

# ❌ Wrong: Variable name inconsistent with Service parameter
async def find_by_id(self, request: web.Request):
    user_id = int(request.match_info["id"])
    user = await self._user_service.find_by_id(id=user_id)  # user_id does not match Service's id parameter
```

#### Service Null Check

Before executing any Service-related operations, you must check if the Service has been initialized. If the Service is not found, log the error and raise a business exception with an error code.

```python
if self._user_service is None:
    message = 'failed to find user_service'
    self._logger.error(message)
    raise Error(UserErrc.SERVICE_NOT_FOUND.value, message)
```

**Notes**:
- Consistent with the DAO Null Check pattern in the Service layer
- Log format uses the `failed to find xxx` pattern
- Must use the feature module's custom error code (e.g., `UserErrc.SERVICE_NOT_FOUND`)

#### Method Ordering

Handler class methods should follow this order:

1. **insert / insert_***: Insert and related methods
2. **update / update_***: Update and related methods (e.g., update_by_id)
3. **delete / delete_***: Delete and related methods (e.g., delete_by_id)
4. **find / find_***: Find and related methods (e.g., find_by_id, find)
5. **register_routes**: Route registration method (at the end)

#### Route Registration Ordering

Route registration order should be consistent with the method order in the Handler example code:

```python
def register_routes(self, app: web.Application):
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
```

#### Parameter Extraction Pattern

When extracting parameters from HTTP requests, follow these conventions:

| Parameter Type | Extraction Method | Processing Rule |
|---------------|-------------------|------------------|
| Path Parameter | `request.match_info["key"]` | Required, directly convert type |
| Query Parameter | `request.query.get("key")` | Optional, use `or None` to handle empty strings |
| Request Body | `await request.json()` | Use `.get()` from JSON, convert empty strings to `None` for optional fields |

```python
# Path parameter
id = int(request.match_info["id"])

# Query parameter (optional)
username = request.query.get("username") or None

# Request body (empty string to None)
request_data = await request.json()
username = request_data.get("username")
if username == "":
    username = None
```

#### Response Format Consistency

All Handler methods should return responses using the unified `SuccessResponse` format, with timestamp using the default value:

```python
# ✅ Correct: Use default timestamp
return web.json_response(
    SuccessResponse(
        code="",
        data=<response_data>
    ).to_dict(),
    status=<HTTP status code>
)

# ❌ Wrong: Manually generate timestamp (SuccessResponse already has default)
timestamp = datetime.utcnow().isoformat() + "Z"
return web.json_response(
    SuccessResponse(
        code="",
        data=<response_data>,
        timestamp=timestamp  # Not needed
    ).to_dict()
)
```

| Operation Type | HTTP Status Code | data Content |
|---------------|-----------------|--------------|
| insert | 201 | Created resource object |
| find / find_by_id | 200 | Resource object or list |
| update_by_id | 200 | Updated resource object |
| delete_by_id | 200 | Empty dict `{}` |

### Dependency Injection Pattern

Handler uses **Setter Injection** pattern for dependency injection:

1. **Constructor injection for required dependencies**:
   - `__init__(config)` - Inject configuration dictionary

2. **Setter method injection for optional dependencies**:
   - `set_user_service(service)` - Inject user service
   - Supports chaining: `handler.set_xxx_service(...).set_yyy_service(...)`

3. **Lazy initialization**:
   - Handler can be created first, dependencies injected later
   - Provides more flexible object lifecycle management

### Handler Initialization Example

```python
from app.feature.user.handler import UserHandler

# Initialize in main.py
user_handler = UserHandler(config=config)
user_handler.set_user_service(service=service)
user_handler.register_routes(app)
```

### Complete UserHandler Template

```python
import logging

from aiohttp import web

from app.common import Error, SuccessResponse, ErrorResponse, Pagination
from app.feature.user import UserField, UserService
from app.feature.user.common import Errc as UserErrc

class UserHandler:
    _logger = logging.getLogger(__name__)
    """User Handler"""

    def __init__(self, config: dict):
        """Initialize

        Args:
            config: Configuration dictionary
        """
        self._config = config
        self._user_service = None

    def set_user_service(self, service: UserService):
        """Set user service

        Args:
            service: User service
        """
        self._user_service = service

    async def insert(self, request: web.Request):
        """Insert user (POST /users)

        Args:
            request: HTTP request

        Returns:
            HTTP response
        """
        try:
            # Parse request body (insert takes JSON body)
            request_data = await request.json()
            username = request_data.get("username")
            password = request_data.get("password")

            # Build UserField object
            user_field = UserField(username=username, password=password)

            # Service null check
            if self._user_service is None:
                message = 'failed to find user_service'
                self._logger.error(message)
                raise Error(UserErrc.SERVICE_NOT_FOUND.value, message)

            # Call service layer
            id = await self._user_service.insert(user=user_field)

            # Return success response
            return web.json_response(
                SuccessResponse(
                    code="",
                    data=id
                ).to_dict(),
                status=201
            )
        except Error as e:
            # Business errors handled by error middleware
            raise
        except Exception as e:
            message = f'failed to process request in insert with username={username}'
            self._logger.error(message)
            raise Error(UserErrc.UNKNOWN_ERROR.value, message) from e

    async def update_by_id(self, request: web.Request):
        """Update user by ID (PUT /users/{id})

        Args:
            request: HTTP request

        Returns:
            HTTP response
        """
        try:
            # Parse path parameter
            id = int(request.match_info["id"])

            # Parse request body (update takes JSON body)
            request_data = await request.json()
            username = request_data.get("username")
            password = request_data.get("password")

            # Empty string means no update for this field
            if username == "":
                username = None
            if password == "":
                password = None

            # Build update params
            update_params = {}
            if username is not None:
                update_params["username"] = username
            if password is not None:
                update_params["password"] = password

            # Service null check
            if self._user_service is None:
                message = 'failed to find user_service'
                self._logger.error(message)
                raise Error(UserErrc.SERVICE_NOT_FOUND.value, message)

            # Call service layer
            user_field = await self._user_service.update_by_id(
                id=id,
                params=update_params
            )

            # Return success response
            return web.json_response(
                SuccessResponse(
                    code="",
                    data=user_field
                ).to_dict()
            )
        except Error as e:
            # Business errors handled by error middleware
            raise
        except Exception as e:
            message = f'failed to process request in update_by_id with id={id}'
            self._logger.error(message)
            raise Error(UserErrc.UNKNOWN_ERROR.value, message) from e

    async def delete_by_id(self, request: web.Request):
        """Delete user by ID (DELETE /users/{id})

        Args:
            request: HTTP request

        Returns:
            HTTP response
        """
        try:
            # Parse path parameter
            id = int(request.match_info["id"])

            # Service null check
            if self._user_service is None:
                message = 'failed to find user_service'
                self._logger.error(message)
                raise Error(UserErrc.SERVICE_NOT_FOUND.value, message)

            # Call service layer
            await self._user_service.delete_by_id(id=id)

            # Return success response (empty data)
            return web.json_response(
                SuccessResponse(
                    code="",
                    data={}
                ).to_dict()
            )
        except Error as e:
            # Business errors handled by error middleware
            raise
        except Exception as e:
            message = f'failed to process request in delete_by_id with id={id}'
            self._logger.error(message)
            raise Error(UserErrc.UNKNOWN_ERROR.value, message) from e

    async def find_by_id(self, request: web.Request):
        """Find user by ID (GET /users/{id})

        Args:
            request: HTTP request

        Returns:
            HTTP response
        """
        try:
            # Parse path parameter
            id = int(request.match_info["id"])

            # Service null check
            if self._user_service is None:
                message = 'failed to find user_service'
                self._logger.error(message)
                raise Error(UserErrc.SERVICE_NOT_FOUND.value, message)

            # Call service layer
            user_field = await self._user_service.find_by_id(id=id)

            # Return success response
            return web.json_response(
                SuccessResponse(
                    code="",
                    data=user_field
                ).to_dict()
            )
        except Error as e:
            # Business errors handled by error middleware
            raise
        except Exception as e:
            message = f'failed to process request in find_by_id with id={id}'
            self._logger.error(message)
            raise Error(UserErrc.UNKNOWN_ERROR.value, message) from e

    async def find(self, request: web.Request):
        """Find user list (GET /users)

        Args:
            request: HTTP request

        Returns:
            HTTP response
        """
        try:
            # Parse query parameters
            username = request.query.get("username") or None
            page_raw = request.query.get("page")
            page_size_raw = request.query.get("page_size")

            # Build query params
            query_params = {"username": username} if username else {}

            # Service null check
            if self._user_service is None:
                message = 'failed to find user_service'
                self._logger.error(message)
                raise Error(UserErrc.SERVICE_NOT_FOUND.value, message)

            # Call service layer
            if page_raw is None and page_size_raw is None:
                user_list, pagination = await self._user_service.find(params=query_params)
            elif page_raw is None:
                user_list, pagination = await self._user_service.find(
                    params=query_params,
                    page_size=max(1, int(page_size_raw))
                )
            elif page_size_raw is None:
                user_list, pagination = await self._user_service.find(
                    params=query_params,
                    page=max(1, int(page_raw))
                )
            else:
                user_list, pagination = await self._user_service.find(
                    params=query_params,
                    page=max(1, int(page_raw)),
                    page_size=max(1, int(page_size_raw))
                )

            # Return success response
            return web.json_response(
                SuccessResponse(
                    code="",
                    data={
                        "items": user_list,
                        "pagination": pagination
                    }
                ).to_dict()
            )
        except Error as e:
            # Business errors handled by error middleware
            raise
        except Exception as e:
            message = f'failed to process request in find with params={query_params}'
            self._logger.error(message)
            raise Error(UserErrc.UNKNOWN_ERROR.value, message) from e

    # Route registration example
    def register_routes(self, app: web.Application):
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
```
