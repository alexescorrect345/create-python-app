## Service Layer

### Module Location

`app/feature/user/service.py`

> **⚠️ Service File Layout**
>
> - When a feature contains one service class, place it in `service.py` regardless of its code size.
> - When a feature contains multiple service classes and each service class has a small amount of code, the classes may be grouped in `service.py`.
> - When multiple service classes are numerous or any service class contains a substantial amount of code, each service class must be placed in its own file, with the file name matching the class name, such as `UserService.py` (e.g., `BvReplayService.py`, `ArchiveReplayService.py`).
> - Each split file should contain one primary Service class.
> - Update `app/feature/{name}/__init__.py` to export all Service classes.

> **⚠️ No-Database Projects**
>
> When no database is configured, omit the DAO import, the `_user_dao` member,
> the `set_user_dao` setter, and all DAO null checks.
> The Service works directly with its data source.

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

Check the DAO at each public method's entry. Private validators rely on this check.

```python
# At the start of insert
if self._user_dao is None:
    message = f'missing user_dao with payload={payload}'
    self._logger.error(message)
    raise Error(CommonErrc.MISSING_DAO.value, message)
```

#### Field Validation Helpers

Reuse `_validate_<field>` methods across insert, update, and find.
Use `def` for pure validation and `async def` when DAO access is needed.

Apply the same validation contract to every field helper:

- Integer fields accept `int` values and integer strings. Reject booleans and
  other types; convert strings with `int()` before applying business rules or
  building DAO parameters.
- Catch conversion failures with `except Exception as e`. Keep the `try` block
  limited to the conversion operation.
- Explicitly invalid values, including type mismatches, blank strings, and
  conversion failures, must be logged with `self._logger.error(message)` before
  checking `allow_none`.
- With `allow_none=True`, an omitted value (`None`) returns `None` directly;
  an invalid value returns `None` after the error log. With `allow_none=False`,
  log and raise the matching `MISSING_*` or `INVALID_*` business error. Use
  `raise Error(...) from e` when wrapping a conversion exception.

```python
async def _validate_username(
    self,
    raw_username: Any,
    check_exist: bool = False,
    allow_none: bool = False,
) -> str | UserField | None:
    """Normalize username, logging invalid input before optionally ignoring it."""
    if raw_username is None:
        if allow_none:
            return None
        message = f'missing username with value={raw_username}'
        self._logger.error(message)
        raise Error(UserErrc.MISSING_USERNAME.value, message)

    if not isinstance(raw_username, str):
        message = f'invalid username with value={raw_username}'
        self._logger.error(message)
        if allow_none:
            return None
        raise Error(UserErrc.INVALID_USERNAME.value, message)

    username = raw_username.strip()
    if not username:
        message = f'invalid username with raw_value={raw_username}'
        self._logger.error(message)
        if allow_none:
            return None
        raise Error(UserErrc.INVALID_USERNAME.value, message)

    if check_exist:
        user_list, _ = await self._user_dao.find({"username": username})
        if user_list:
            return user_list[0]

    return username
```

Public methods decide whether a matching record represents a conflict:

```python
# insert: any matching username is a conflict
res_username = await self._validate_username(
    raw_username=payload.get("username"),
    check_exist=True,
)
if isinstance(res_username, UserField):
    message = f'username already exists with existing_user_id={res_username.id}'
    self._logger.error(message)
    raise Error(UserErrc.EXIST_USERNAME.value, message)
username = res_username
```

```python
# update_by_id: the current record's username is allowed
if "username" in payload:
    res_username = await self._validate_username(
        raw_username=payload["username"],
        check_exist=True,
    )
    if isinstance(res_username, UserField):
        if res_username.id != id:
            message = f'username already exists with id={id}, existing_user_id={res_username.id}'
            self._logger.error(message)
            raise Error(UserErrc.EXIST_USERNAME.value, message)
        username = res_username.username
    else:
        username = res_username

    if username != user_field.username:
        params["username"] = username
```

```python
# find: omitted usernames are ignored; invalid usernames are logged and ignored
username = await self._validate_username(
    raw_username=payload.get("username"),
    allow_none=True,
)
if username is not None:
    params["username"] = username
```

#### Class Layout and Method Ordering

Place the class docstring first, followed by the class-level logger:

```python
class UserService:
    """User Service"""

    _logger = logging.getLogger(__name__)
```

Service class methods should follow this order:

1. **__init__**: Store configuration and initialize dependency members
2. **set_***: Dependency setter methods, such as set_user_dao
3. **insert / insert_***: Insert and related methods
4. **upsert / upsert_***: Upsert and related methods (DolphinDB-specific and optional, e.g., upsert_by_id)
5. **update / update_***: Update and related methods (e.g., update_by_id)
6. **delete / delete_***: Delete and related methods (e.g., delete_by_id)
7. **find_by_id**, then **find** and other query methods
8. **Private helper methods**: Place field validators such as _validate_username after all public methods

This order presents dependency setup first, business operations next, and reusable validation details last.

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
from app.feature.user import UserDao, UserService

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
from typing import Any

from app.common import Errc as CommonErrc, Error, Pagination
from app.feature.user.common import Errc as UserErrc, FieldType
from app.feature.user.dao import UserDao
from app.feature.user.field import UserField


class UserService:
    """User Service"""

    _logger = logging.getLogger(__name__)

    def __init__(self, config: dict[str, Any]) -> None:
        """Initialize

        Args:
            config: Configuration dictionary
        """
        self._config = config
        self._user_dao: UserDao | None = None

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
                - role_id: Role ID, as an integer or integer string (required)
                - username: Username (required)
                - password: Password (required)

        Returns:
            User ID

        Raises:
            Error: MISSING_DAO, MISSING_ROLE_ID, INVALID_ROLE_ID,
                MISSING_USERNAME, INVALID_USERNAME, EXIST_USERNAME,
                MISSING_PASSWORD, INVALID_PASSWORD, or an error from the DAO
        """
        if self._user_dao is None:
            message = f'missing user_dao with payload={payload}'
            self._logger.error(message)
            raise Error(CommonErrc.MISSING_DAO.value, message)

        # Validate role_id
        role_id = self._validate_role_id(raw_role_id=payload.get("role_id"))

        # Validate username
        res_username = await self._validate_username(
            raw_username=payload.get("username"),
            check_exist=True,
        )
        if isinstance(res_username, UserField):
            message = f'username already exists with existing_user_id={res_username.id}'
            self._logger.error(message)
            raise Error(UserErrc.EXIST_USERNAME.value, message)
        username = res_username

        # Validate password
        password = self._validate_password(raw_password=payload.get("password"))

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
                - role_id: New role ID, as an integer or integer string (optional)
                - username: New username (optional)
                - password: New password (optional)

        Returns:
            None

        Raises:
            Error: MISSING_DAO, MISSING_FIELD, MISSING_ROLE_ID, INVALID_ROLE_ID,
                MISSING_USERNAME, INVALID_USERNAME, EXIST_USERNAME,
                MISSING_PASSWORD, INVALID_PASSWORD, or an error from the DAO
        """
        # DAO null check
        if self._user_dao is None:
            message = f'missing user_dao with id={id}, payload={payload}'
            self._logger.error(message)
            raise Error(CommonErrc.MISSING_DAO.value, message)

        if not payload:
            self._logger.warning(f'empty payload for update user with id={id}')
            return None

        # Check if user exists
        user_field = await self._user_dao.find_by_id(id)
        if not user_field:
            message = f'missing user_field with id={id}, payload={payload}'
            self._logger.error(message)
            raise Error(CommonErrc.MISSING_FIELD.value, message)

        params: dict[str, Any] = {}

        # Validate role_id
        if "role_id" in payload:
            role_id = self._validate_role_id(raw_role_id=payload["role_id"])
            if role_id != user_field.role_id:
                params["role_id"] = role_id

        # Validate username
        if "username" in payload:
            res_username = await self._validate_username(
                raw_username=payload["username"],
                check_exist=True,
            )
            if isinstance(res_username, UserField):
                if res_username.id != id:
                    message = f'username already exists with id={id}, existing_user_id={res_username.id}'
                    self._logger.error(message)
                    raise Error(UserErrc.EXIST_USERNAME.value, message)
                username = res_username.username
            else:
                username = res_username

            if username != user_field.username:
                params["username"] = username

        # Validate password
        if "password" in payload:
            password = self._validate_password(raw_password=payload["password"])
            if password != user_field.password:
                params["password"] = password

        if not params:
            self._logger.info(f'no user fields changed with id={id}, payload={payload}')
            return None

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

    async def find_by_id(self, id: int, field_type: FieldType = FieldType.SIMPLE) -> UserField | None:
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
        orderby: list[tuple[str, str]] | None = None,
        field_type: FieldType = FieldType.SIMPLE,
        page: int = 1,
        page_size: int = sys.maxsize
    ) -> tuple[list[UserField], Pagination]:
        """Find users

        Args:
            payload: Query parameter dictionary, supported keys:
                - role_id: Integer role ID or integer string (optional;
                    invalid values are logged and ignored)
                - username: Username (optional;
                    blank or invalid values are logged and ignored)
            orderby: Order spec list, each item is a (field_name, direction) tuple;
                direction must be 'asc' or 'desc'; order follows list order,
                defaults to [('id', 'desc')] when None or empty
            field_type: Query type (FieldType.SIMPLE or FieldType.FULL)
            page: Page number (starting from 1)
            page_size: Number of items per page

        Returns:
            User field object list with pagination info

        Raises:
            Error: MISSING_DAO, INVALID_PAGE, INVALID_PAGE_SIZE,
                or an error from the DAO
        """
        # DAO null check
        if self._user_dao is None:
            message = f'missing user_dao with payload={payload}, orderby={orderby}, field_type={field_type}, page={page}, page_size={page_size}'
            self._logger.error(message)
            raise Error(CommonErrc.MISSING_DAO.value, message)

        if page < 1:
            message = f'invalid page with page={page}'
            self._logger.error(message)
            raise Error(CommonErrc.INVALID_PAGE.value, message)

        if page_size < 1:
            message = f'invalid page_size with page_size={page_size}'
            self._logger.error(message)
            raise Error(CommonErrc.INVALID_PAGE_SIZE.value, message)

        params: dict[str, Any] = {}

        # Validate role_id
        role_id = self._validate_role_id(
            raw_role_id=payload.get("role_id"),
            allow_none=True,
        )
        if role_id is not None:
            params["role_id"] = role_id

        # Validate username
        username = await self._validate_username(
            raw_username=payload.get("username"),
            allow_none=True,
        )
        if username is not None:
            params["username"] = username

        return await self._user_dao.find(
            params=params,
            orderby=orderby,
            page=page,
            page_size=page_size,
            field_type=field_type
        )

    def _validate_role_id(
        self,
        raw_role_id: Any,
        allow_none: bool = False,
    ) -> int | None:
        """Validate and normalize a role ID.

        Args:
            raw_role_id: Unvalidated role ID, as an integer or integer string
            allow_none: Whether missing or invalid values are ignored;
                invalid values are logged before being ignored

        Returns:
            Normalized integer role ID, or None when optional input is ignored

        Raises:
            Error: MISSING_ROLE_ID or INVALID_ROLE_ID
        """
        if raw_role_id is None:
            if allow_none:
                return None
            message = f'missing role_id with value={raw_role_id}'
            self._logger.error(message)
            raise Error(UserErrc.MISSING_ROLE_ID.value, message)

        if isinstance(raw_role_id, bool) or not isinstance(raw_role_id, (int, str)):
            message = f'invalid role_id with value={raw_role_id}'
            self._logger.error(message)
            if allow_none:
                return None
            raise Error(UserErrc.INVALID_ROLE_ID.value, message)

        try:
            role_id = int(raw_role_id) if isinstance(raw_role_id, str) else raw_role_id
        except Exception as e:
            message = f'invalid role_id with value={raw_role_id}'
            self._logger.error(message)
            if allow_none:
                return None
            raise Error(UserErrc.INVALID_ROLE_ID.value, message) from e

        return role_id

    async def _validate_username(
        self,
        raw_username: Any,
        check_exist: bool = False,
        allow_none: bool = False,
    ) -> str | UserField | None:
        """Validate and normalize a username, optionally checking for a match.

        Args:
            raw_username: Unvalidated username
            check_exist: Whether to find a user with the normalized username
            allow_none: Whether missing or invalid values are ignored;
                invalid values are logged before being ignored

        Returns:
            Normalized username, matching UserField, or None when ignored

        Raises:
            Error: MISSING_USERNAME, INVALID_USERNAME, or an error from the DAO
        """
        if raw_username is None:
            if allow_none:
                return None
            message = f'missing username with value={raw_username}'
            self._logger.error(message)
            raise Error(UserErrc.MISSING_USERNAME.value, message)

        if not isinstance(raw_username, str):
            message = f'invalid username with value={raw_username}'
            self._logger.error(message)
            if allow_none:
                return None
            raise Error(UserErrc.INVALID_USERNAME.value, message)

        username = raw_username.strip()
        if not username:
            message = f'invalid username with raw_value={raw_username}'
            self._logger.error(message)
            if allow_none:
                return None
            raise Error(UserErrc.INVALID_USERNAME.value, message)

        if check_exist:
            user_list, _ = await self._user_dao.find({"username": username})
            if user_list:
                return user_list[0]

        return username

    def _validate_password(self, raw_password: Any) -> str:
        """Validate a required password string, preserving whitespace.

        Args:
            raw_password: Unvalidated password

        Returns:
            Non-empty password string with its original whitespace

        Raises:
            Error: MISSING_PASSWORD or INVALID_PASSWORD
        """
        if raw_password is None:
            message = 'missing password'
            self._logger.error(message)
            raise Error(UserErrc.MISSING_PASSWORD.value, message)

        if not isinstance(raw_password, str):
            message = 'invalid password'
            self._logger.error(message)
            raise Error(UserErrc.INVALID_PASSWORD.value, message)

        password = raw_password
        if not password:
            message = 'invalid password'
            self._logger.error(message)
            raise Error(UserErrc.INVALID_PASSWORD.value, message)

        return password
```
