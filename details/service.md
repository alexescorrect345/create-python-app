## Service Layer

### Module Location

`app/feature/user/service.py`

### Service Layer Responsibilities

1. **Business Logic**: Validate data legality, process business rules
2. **Call DAO**: Access database through DAO
3. **Return Field Objects**: Convert raw data to Field objects
4. **Throw Business Errors**: Use `Errc` error codes
5. **Stay DB-Agnostic**: All database-specific details (SQL vs DolphinDB scripts, parameter binding, and ID generation) are owned by the DAO, so the Service is identical across SQLite and DolphinDB

### Rules

#### Parameter Naming Consistency with DAO

If function parameters, local variables, or DAO method return values are associated with DAO method parameters, their naming should be **exactly consistent** with the DAO method parameters.

For example:
- DAO method `find_by_id(self, id: int)` uses parameter name `id`
- Then Service method `find_by_id` should also use parameter name `id`, not `user_id`

This ensures:
1. Code consistency and readability
2. Clear data flow tracing
3. Reduced cognitive load when switching between layers

```python
# ✅ Correct: Consistent with DAO parameter naming
async def find_by_id(self, id: int) -> UserField:
    user_field = await self._user_dao.find_by_id(id)

# ❌ Wrong: Inconsistent parameter naming
async def find_by_id(self, user_id: int) -> UserField:
    user_field = await self._user_dao.find_by_id(user_id)
```

#### DAO Null Check

Before executing all DAO-related operations, must check if DAO is initialized. If DAO is not initialized, should throw a business error.

```python
if self._user_dao is None:
    message = f'missing user_dao with <params>={}'
    self._logger.error(message)
    raise Error(CommonErrc.MISSING_DAO.value, message)
```

#### Method Ordering

The order of methods in a Service class should follow this sequence:

1. **insert / insert_***: Insert and related methods
2. **upsert / upsert_***: Upsert and related methods (DolphinDB-specific and optional, e.g., upsert_by_id)
3. **update / update_***: Update and related methods (e.g., update_by_id)
4. **delete / delete_***: Delete and related methods (e.g., delete_by_id)
5. **find / find_***: Find and related methods (e.g., find_by_id)

This order ensures a logical flow from data manipulation to data retrieval.

Skip methods that are not provided by the selected DAO. For example, `upsert` usually only exists in DolphinDB-based DAOs.

### Dependency Injection Pattern

Service uses **Setter Injection** pattern to inject dependencies:

1. **Constructor injection for required dependencies**:
   - `__init__(config)` - Inject configuration dictionary

2. **Setter method injection for optional dependencies**:
   - `set_user_dao(user_dao)` - Inject user data access object

3. **Lazy initialization**:
   - Service can be created first, DAO dependency injected later
   - Provides more flexible object lifecycle management

### Service Initialization Example

```python
from app.feature.user.dao import UserDao
from app.feature.user.service import UserService

# Initialize in main.py
user_service = UserService(config=config)
user_service.set_user_dao(user_dao=user_dao)
```

> **Note**: The `id` type in this template uses `int` as an example. In practice, the `id` type depends on the business requirements and database design (e.g., `int`, `str`, etc.).

### Complete UserService Template (Unified)

> This template works unchanged on both SQLite and DolphinDB, because the DAO owns all database-specific details (including ID generation). For DolphinDB-only DAO methods (such as `upsert` and `batch_insert`), add corresponding Service methods that mirror their signatures; see `dao.md`.

```python
import sys
import logging
from typing import Any, Optional

from app.common import Error, Errc as CommonErrc, Pagination
from app.feature.user.common import Errc as UserErrc, FieldType
from app.feature.user.dao import UserDao
from app.feature.user.field import UserField

class UserService:
    """User Service"""

    _logger = logging.getLogger(__name__)

    def __init__(self, config: dict[str, Any]):
        """Initialize

        Args:
            config: Configuration dictionary
        """
        self._config = config
        self._user_dao: Optional[UserDao] = None

    def set_user_dao(self, user_dao: UserDao) -> None:
        """Set user data access object

        Args:
            user_dao: User data access object
        """
        self._user_dao = user_dao

    async def insert(self, payload: dict[str, Any]) -> Any:
        """Insert user

        Args:
            payload: User parameter dictionary with keys:
                - role_id: Role ID (required)
                - username: Username (required)
                - password: Password (required)

        Returns:
            User ID

        Raises:
            Error: MISSING_DAO, MISSING_ROLE_ID, MISSING_USERNAME, MISSING_PASSWORD
        """
        if self._user_dao is None:
            message = f'missing user_dao with payload={payload}'
            self._logger.error(message)
            raise Error(CommonErrc.MISSING_DAO.value, message)

        # handle role_id
        raw_role_id = payload.get("role_id")
        if raw_role_id is None:
            message = f'missing role_id with payload={payload}'
            self._logger.error(message)
            raise Error(UserErrc.MISSING_ROLE_ID.value, message)
        try:
            role_id = int(raw_role_id)
        except Exception as e:
            message = f'invalid role_id with value={raw_role_id}'
            self._logger.error(message)
            raise Error(UserErrc.INVALID_ROLE_ID.value, message) from e

        # handle username
        raw_username = payload.get("username")
        if not raw_username:
            message = f'missing username with payload={payload}'
            self._logger.error(message)
            raise Error(UserErrc.MISSING_USERNAME.value, message)
        username = str(raw_username)

        # handle password
        raw_password = payload.get("password")
        if not raw_password:
            message = f'missing password with payload={payload}'
            self._logger.error(message)
            raise Error(UserErrc.MISSING_PASSWORD.value, message)
        password = str(raw_password)

        # id is intentionally omitted; it is generated by the DAO
        # (SQLite via AUTOINCREMENT, DolphinDB via max(id)+1) and returned.
        user_field = UserField(
            role_id=role_id,
            username=username,
            password=password
        )

        id = await self._user_dao.insert(user_field)
        self._logger.info(f'succeeded to insert user with id={id}, user_field={user_field}')

        return id

    async def update_by_id(self, id: int, payload: dict[str, Any]) -> None:
        """Update user by ID

        Args:
            id: User ID
            payload: Update parameter dictionary, supported keys:
                - role_id: New role ID (optional)
                - username: New username (optional)
                - password: New password (optional)

        Returns:
            None

        Raises:
            Error: MISSING_DAO, MISSING_USER_FIELD, INVALID_ROLE_ID
        """
        # DAO null check
        if self._user_dao is None:
            message = f'missing user_dao with id={id}, payload={payload}'
            self._logger.error(message)
            raise Error(CommonErrc.MISSING_DAO.value, message)

        # Check if user exists
        user_field = await self._user_dao.find_by_id(id)
        if not user_field:
            message = f'missing user_field with id={id}, payload={payload}'
            self._logger.error(message)
            raise Error(CommonErrc.MISSING_FIELD.value, message)

        if not payload:
            self._logger.warning(f'empty payload for update user with id={id}')
            return None

        params: dict[str, Any] = {}

        # handle role_id
        if "role_id" in payload:
            raw_role_id = payload["role_id"]
            try:
                params["role_id"] = int(raw_role_id)
            except Exception as e:
                message = f'invalid role_id with value={raw_role_id}'
                self._logger.error(message)
                raise Error(UserErrc.INVALID_ROLE_ID.value, message) from e

        # handle username
        if "username" in payload:
            raw_username = payload["username"]
            if not raw_username:
                message = f'missing username with payload={payload}'
                self._logger.error(message)
                raise Error(UserErrc.MISSING_USERNAME.value, message)
            params["username"] = str(raw_username)

        # handle password
        if "password" in payload:
            raw_password = payload["password"]
            if not raw_password:
                message = f'missing password with payload={payload}'
                self._logger.error(message)
                raise Error(UserErrc.MISSING_PASSWORD.value, message)
            params["password"] = str(raw_password)

        # Update user
        await self._user_dao.update_by_id(id, params=params)
        self._logger.info(f'succeeded to update user with id={id}, params={params}')

        return None

    async def delete_by_id(self, id: int) -> None:
        """Delete user by ID

        Args:
            id: User ID

        Raises:
            Error: MISSING_DAO
        """
        # DAO null check
        if self._user_dao is None:
            message = f'missing user_dao with id={id}'
            self._logger.error(message)
            raise Error(CommonErrc.MISSING_DAO.value, message)

        # Delete user
        await self._user_dao.delete_by_id(id)
        self._logger.info(f'succeeded to delete user with id={id}')

    async def find_by_id(self, id: int, field_type: FieldType = FieldType.SIMPLE) -> Optional[UserField]:
        """Find user by ID

        Args:
            id: User ID
            field_type: Query type (FieldType.SIMPLE or FieldType.FULL)

        Returns:
            User field object or None

        Raises:
            Error: MISSING_DAO
        """
        # DAO null check
        if self._user_dao is None:
            message = f'missing user_dao with id={id}'
            self._logger.error(message)
            raise Error(CommonErrc.MISSING_DAO.value, message)

        return await self._user_dao.find_by_id(id, field_type)

    async def find(
        self,
        payload: dict[str, Any],
        orderby: Optional[list[tuple[str, str]]] = None,
        field_type: FieldType = FieldType.SIMPLE,
        page: int = 1,
        page_size: int = sys.maxsize
    ) -> tuple[list[UserField], Pagination]:
        """Find users

        Args:
            payload: Query parameter dictionary, supported keys:
                - role_id: Role ID (optional)
                - username: Username (optional)
            orderby: Order spec list, each item is a (field_name, direction) tuple;
                direction must be 'asc' or 'desc'; order follows list order,
                defaults to [('id', 'desc')] when None or empty
            field_type: Query type (FieldType.SIMPLE or FieldType.FULL)
            page: Page number (starting from 1)
            page_size: Number of items per page

        Returns:
            User field object list with pagination info

        Raises:
            Error: MISSING_DAO, INVALID_ROLE_ID
        """
        # DAO null check
        if self._user_dao is None:
            message = f'missing user_dao with payload={payload}, orderby={orderby}, field_type={field_type}, page={page}, page_size={page_size}'
            self._logger.error(message)
            raise Error(CommonErrc.MISSING_DAO.value, message)

        params: dict[str, Any] = {}

        # handle role_id
        if "role_id" in payload:
            raw_role_id = payload["role_id"]
            try:
                params["role_id"] = int(raw_role_id)
            except Exception as e:
                message = f'invalid role_id with value={raw_role_id}'
                self._logger.error(message)
                raise Error(UserErrc.INVALID_ROLE_ID.value, message) from e

        # handle username
        if "username" in payload:
            raw_username = payload["username"]
            if raw_username and str(raw_username).strip():
                params["username"] = str(raw_username).strip()

        return await self._user_dao.find(
            params=params,
            orderby=orderby,
            page=page,
            page_size=page_size,
            field_type=field_type
        )
```
