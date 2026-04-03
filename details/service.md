## Service Layer

### Module Location

`app/feature/user/service.py`

### Service Layer Responsibilities

1. **Business Logic**: Validate data legality, process business rules
2. **Call DAO**: Access database through DAO
3. **Return Field Objects**: Convert raw data to Field objects
4. **Throw Business Errors**: Use `Errc` error codes

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
    message = f'failed to find user_dao in UserService'
    self._logger.error(message)
    raise Error(str(CommonErrc.MISSING_DAO), message)
```

#### Method Ordering

The order of methods in a Service class should follow this sequence:

1. **insert / insert_***: Insert and related methods
2. **update / update_***: Update and related methods (e.g., update_by_id)
3. **delete / delete_***: Delete and related methods (e.g., delete_by_id)
4. **find / find_***: Find and related methods (e.g., find_by_id)

This order ensures a logical flow from data manipulation to data retrieval.

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

### Complete UserService Template

```python
import sys
import logging

from app.common import Error, Pagination, Errc as CommonErrc
from app.feature.user.common import Errc as UserErrc
from app.feature.user.dao import UserDao
from app.feature.user.field import UserField

class UserService:
    """User Service"""
    _logger = logging.getLogger(__name__)

    def __init__(self, config: dict):
        """Initialize

        Args:
            config: Configuration dictionary
        """
        self._config = config
        self._user_dao = None

    def set_user_dao(self, user_dao: UserDao):
        """Set user data access object

        Args:
            user_dao: User data access object
        """
        self._user_dao = user_dao

    async def insert(self, user_field: UserField) -> int:
        """Insert user

        Args:
            user_field: User field object

        Returns:
            User ID
        """
        # DAO null check
        if self._user_dao is None:
            message = f'failed to find user_dao in {self.__class__.__name__}'
            self._logger.error(message)
            raise Error(str(CommonErrc.MISSING_DAO), message)

        # Business logic: validate required fields
        if not user_field.username:
            message = f'failed to validate user with username={user_field.username}'
            self._logger.error(message)
            raise Error(str(UserErrc.USERNAME_REQUIRED), message)

        # Business logic: check if username already exists
        existing_user_field = await self._user_dao.find_by_username(username=user_field.username)
        if existing_user_field:
            message = f'failed to insert user with username={user_field.username}, username already exists'
            self._logger.error(message)
            raise Error(str(UserErrc.USERNAME_EXISTS), message)

        # Create user
        id = await self._user_dao.insert(user=user_field)
        self._logger.info(f'succeeded to insert user with username={user_field.username}, id={id}')

        # Return user ID
        return id

    async def update_by_id(self, id: int, params: dict) -> UserField:
        """Update user by ID

        Args:
            id: User ID
            params: Update parameter dictionary, supported keys:
                - username: New username (optional)
                - password: New password (optional)

        Returns:
            Updated user field object
        """
        # DAO null check
        if self._user_dao is None:
            message = f'failed to find user_dao in {self.__class__.__name__}'
            self._logger.error(message)
            raise Error(str(CommonErrc.MISSING_DAO), message)

        # Check if user exists
        user_field = await self._user_dao.find_by_id(id)
        if not user_field:
            message = f'failed to find user by id={id}'
            self._logger.error(message)
            raise Error(str(CommonErrc.MISSING_FIELD), message)

        # If updating username, check if new username already exists
        username = params.get("username")
        if username and username != user_field.username:
            user_list = await self._user_dao.find(params={"username": username})
            if user_list[0]:
                message = f'failed to update user with username={username}, username already exists'
                self._logger.error(message)
                raise Error(str(UserErrc.USERNAME_EXISTS), message)

        # Update user
        await self._user_dao.update_by_id(id, params=params)
        self._logger.info(f'succeeded to update user with id={id}, params={params}')

        # Return updated user
        user_field = await self._user_dao.find_by_id(id)
        return user_field

    async def delete_by_id(self, id: int) -> None:
        """Delete user by ID

        Args:
            id: User ID
        """
        # DAO null check
        if self._user_dao is None:
            message = f'failed to find user_dao in {self.__class__.__name__}'
            self._logger.error(message)
            raise Error(str(CommonErrc.MISSING_DAO), message)

        # Check if user exists
        user_field = await self._user_dao.find_by_id(id)
        if not user_field:
            message = f'failed to find user by id={id}'
            self._logger.error(message)
            raise Error(str(CommonErrc.MISSING_FIELD), message)

        # Delete user
        await self._user_dao.delete_by_id(id)
        self._logger.info(f'succeeded to delete user with id={id}')

    async def find_by_id(self, id: int) -> UserField:
        """Find user by ID

        Args:
            id: User ID

        Returns:
            User field object
        """
        # DAO null check
        if self._user_dao is None:
            message = f'failed to find user_dao in {self.__class__.__name__}'
            self._logger.error(message)
            raise Error(str(CommonErrc.MISSING_DAO), message)

        # Query user
        user_field = await self._user_dao.find_by_id(id)
        if not user_field:
            message = f'failed to find user by id={id}'
            self._logger.error(message)
            raise Error(str(CommonErrc.MISSING_FIELD), message)

        return user_field

    async def find(
        self,
        params: dict,
        page: int = 1,
        page_size: int = sys.maxsize
    ) -> tuple[list[UserField], Pagination]:
        """Find users

        Args:
            params: Query parameter dictionary, supported keys:
                - username: Username (optional)
            page: Page number (starting from 1)
            page_size: Number of items per page

        Returns:
            User field object list with pagination info
        """
        # DAO null check
        if self._user_dao is None:
            message = f'failed to find user_dao in {self.__class__.__name__}'
            self._logger.error(message)
            raise Error(str(CommonErrc.MISSING_DAO), message)

        user_list, pagination = await self._user_dao.find(
            params=params,
            page=page,
            page_size=page_size
        )
        return user_list, pagination
```
