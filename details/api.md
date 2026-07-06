## API Layer

### Module Location

`app/api/{name}Api.py`

### API Layer Responsibilities

1. **HTTP Client**: Call external REST APIs via aiohttp
2. **Request Building**: Construct URLs, headers, params, payloads
3. **Response Handling**: Parse API responses, extract data
4. **Error Handling**: Handle HTTP errors and network exceptions
5. **Logging**: Log request/response for debugging

### Two Types of API Classes

#### Type 1: InternalApi

`InternalApi` is a base class defined in `app/api/api.py` (see Step 8 in SKILL.md), providing `_get`, `_post`, `_put`, `_delete` methods that encapsulate HTTP requests for `SuccessResponse`/`ErrorResponse` format APIs. All internal API clients must subclass `InternalApi` to implement interface access.

**Characteristics**:
- Response format is `SuccessResponse` or `ErrorResponse`
- Base class methods automatically check the `code` field: empty string means success, non-empty means error
- On success, return the complete `SuccessResponse` object (caller accesses `.data` for payload)
- On error, throw an `Error` exception

#### Type 2: ExternalApi

`ExternalApi` is a base class defined in `app/api/api.py` (see Step 8 in SKILL.md), providing `_get`, `_post`, `_put`, `_delete` methods that encapsulate HTTP requests and return raw JSON responses. All external API clients must subclass `ExternalApi` to implement interface access.

**Characteristics**:
- Directly return raw `response.json()`
- Need to check HTTP status code
- Caller parses the response structure on their own

### Rules

#### Method Ordering

The order of methods in an API class should follow this sequence:

1. **insert / insert_***: Create and related methods
2. **update / update_***: Update and related methods (e.g., update_user_by_id)
3. **delete / delete_***: Delete and related methods (e.g., delete_user_by_id)
4. **find / find_***: Query and related methods (e.g., find_user_by_id, find_user)

#### Method Naming

Method naming must include the entity name, format: `{action}_{entity}` or `{action}_{entity}_by_{field}`.

**Correct examples**:
- `insert_user` - Insert a user
- `update_user_by_id` - Update user by ID
- `delete_user_by_id` - Delete user by ID
- `find_user_by_id` - Find user by ID
- `find_user` - Find user list

**Incorrect examples**:
- `insert` - Missing entity name
- `update_by_id` - Missing entity name
- `find` - Missing entity name

**ExternalApi Method Naming**:

ExternalApi typically doesn't have a complete CRUD system, only one query method, should use `fetch` instead of `find`:

- `fetch` - Query (for external APIs without complete CRUD)

#### Config

Configuration rules in `config.toml` and `config.dev.toml`:

```toml
[service.api.{name}]
base_url = "http://localhost:8000"
timeout_s = 5
```

Where `{name}` comes from the `{name}Api.py` filename.

### Complete Derived InternalApi Template

All InternalApi subclasses inherit from `app.api.api.InternalApi` base class. Subclasses only need to implement business methods by calling `self._get()`, `self._post()`, `self._put()`, `self._delete()`.

```python
from typing import Any, Dict, Optional

from app.common import SuccessResponse
from app.api.api import InternalApi


class UserApi(InternalApi):
    """User API Client

    Calls internal user APIs created by this project's skills
    """

    async def insert_user(
        self,
        role_id: int,
        username: str,
        password: str
    ) -> SuccessResponse:
        """Insert a user

        Args:
            role_id: Role ID
            username: Username
            password: Password

        Returns:
            SuccessResponse object (caller accesses .data["id"] for the created user ID)

        Raises:
            Error: API request failed
        """
        return await self._post('/users', {
            "role_id": role_id,
            "username": username,
            "password": password
        })

    async def update_user_by_id(self, id: int, params: Dict[str, Any]) -> SuccessResponse:
        """Update user by ID

        Args:
            id: User ID
            params: Update params, e.g. {"username": "new_name"}

        Returns:
            SuccessResponse object (caller accesses .data for response payload)

        Raises:
            Error: API request failed
        """
        return await self._put(f'/users/{id}', params)

    async def delete_user_by_id(self, id: int) -> None:
        """Delete user by ID

        Args:
            id: User ID

        Raises:
            Error: API request failed
        """
        await self._delete(f'/users/{id}')

    async def find_user_by_id(self, id: int, field_type: str = 'simple') -> SuccessResponse:
        """Find user by ID

        Args:
            id: User ID
            field_type: Query type, 'simple' (default) or 'full'

        Returns:
            SuccessResponse object (caller accesses .data for user info)

        Raises:
            Error: API request failed
        """
        return await self._get(f'/users/{id}', params={"field_type": field_type})

    async def find_user(
        self,
        params: Dict[str, Any],
        orderby: Optional[str] = None,
        field_type: Optional[str] = None,
        page: Optional[int] = None,
        page_size: Optional[int] = None
    ) -> SuccessResponse:
        """Find user list

        Args:
            params: Query filters
                - role_id: Filter by role ID (optional)
                - username: Filter by username (optional)
            orderby: Order spec, e.g. 'id:desc,username:asc' (optional)
            field_type: 'simple' (default) or 'full' (optional)
            page: Page number starting from 1 (optional)
            page_size: Items per page (optional)

        Returns:
            SuccessResponse object (caller accesses .data for user list)

        Raises:
            Error: API request failed
        """
        query = dict(params)
        if orderby is not None:
            query["orderby"] = orderby
        if field_type is not None:
            query["field_type"] = field_type
        if page is not None:
            query["page"] = page
        if page_size is not None:
            query["page_size"] = page_size
        return await self._get('/users', params=query)
```

### Complete Derived ExternalApi Template

All ExternalApi subclasses inherit from `app.api.api.ExternalApi` base class. Subclasses only need to implement business methods by calling `self._get()`, `self._post()`, `self._put()`, `self._delete()`.

```python
from typing import Any, Dict

from app.api.api import ExternalApi


class MyExternalApi(ExternalApi):
    """MyExternal API Client

    Calls third-party APIs
    Returns raw JSON, caller parses response structure on their own
    """

    async def fetch(
        self,
        params: Dict[str, Any]
    ) -> Dict[str, Any]:
        """Query

        Args:
            params: Query params

        Returns:
            Raw JSON response, structure defined by third-party API

        Raises:
            Error: HTTP request failed
        """
        return await self._get('/data', params=params)
```
