## DAO Layer

### Module Location

`app/feature/user/dao.py`

### DAO Layer Responsibilities

1. **Only responsible for database operations**: Execute SQL queries, return raw data
2. **No business logic handling**: No data validation, no business error throwing
3. **Throw database errors**: Only throw DB errors
4. **Parameter validation**: Not validated at DAO layer (handled by Service layer)

### Rules

#### Method Ordering

The order of methods in a DAO class should follow this sequence:

1. **init_table**: Table initialization
2. **insert / insert_***: Insert and related methods (e.g., insert, insert_batch)
3. **update / update_***: Update and related methods (e.g., update_by_id, update)
4. **delete / delete_***: Delete and related methods (e.g., delete_by_id, delete)
5. **find / find_***: Find and related methods (e.g., find_by_id, find_by_username, find)

This order ensures a logical flow from table setup → data manipulation → data retrieval.

#### SQL String Conventions

- Use **single quotes** for SQL statements or other database query script strings
- When the statement itself contains single quotes, use **double quotes**
- For SQL scripts that need line breaks, use **three single quotes** (`'''`)
- Query script fragments used in concatenation (such as column names, condition fragments, etc.) must also follow the rules above

```python
# ✅ Correct: Use single quotes
sql = f'SELECT id, username, password FROM {self._TABLE_NAME} WHERE id = ?'
sql = f'INSERT INTO {self._TABLE_NAME} (username, password) VALUES (?, ?) RETURNING id'
updates.append('username = ?')

# ✅ Correct: Use three single quotes for line breaks
sql = f'''
CREATE TABLE IF NOT EXISTS {self._TABLE_NAME} (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT NOT NULL UNIQUE,
    password TEXT NOT NULL
)
'''

# ✅ Correct: Use double quotes when statement itself contains single quotes
sql = f"SELECT * FROM {self._TABLE_NAME} WHERE username = '{username}'"

# ❌ Wrong: Use double quotes
sql = f"SELECT id, username, password FROM {self._TABLE_NAME} WHERE id = ?"
# ❌ Wrong: Use three double quotes for line breaks
sql = f"""
CREATE TABLE IF NOT EXISTS {self._TABLE_NAME} (
    id INTEGER PRIMARY KEY AUTOINCREMENT
)
"""
```

#### Variable Naming for Database Fields

- If a variable represents a field in a database record, the variable naming should match the database field name (when no conflict exists)
- For example: If the primary key field of table `t_user` is `id`, use `id` instead of `user_id` as the variable name

```python
# ✅ Correct: Use variable name matching the database field
sql = f'SELECT id, username, password FROM {self._TABLE_NAME} WHERE id = ?'
rows = await self._db.exec(sql, (id,))

# ❌ Wrong: Use user_id instead of id
sql = f'SELECT id, username, password FROM {self._TABLE_NAME} WHERE id = ?'
rows = await self._db.exec(sql, (user_id,))  # Should use id
```

### Dependency Injection Pattern

DAO uses **Setter Injection** pattern for dependency injection:

1. **Constructor injection for required dependencies**:
   - `__init__(config)` - Inject configuration dictionary

2. **Setter method injection for optional dependencies**:
   - `set_db(db)` - Inject database instance
   - Supports chaining: `dao.set_db(db)`

3. **Lazy initialization**:
   - DAO can be created first, DB dependency injected later
   - Provides more flexible object lifecycle management

### DAO Initialization Example

```python
from app.feature.user.dao import UserDao

# Initialize in main.py
user_dao = UserDao(config=config)
user_dao.set_db(db=db)
```

### Complete UserDao Template

> ⚠️ The following example is based on `SqliteDB` implementation. Adjust accordingly when using other databases.

```python
import sys
import logging

from app.common import Error, Pagination
from app.db.common import Errc as DbErrc
from app.db import DB
from app.feature.user import UserField

class UserDao:
    """User Data Access Object"""

    _TABLE_NAME = 't_user'
    _logger = logging.getLogger(__name__)

    def __init__(self, config: dict):
        """Initialize

        Args:
            config: Configuration dictionary
        """
        self._config = config
        self._db = None

    def set_db(self, db: DB):
        """Set database instance

        Args:
            db: Database instance
        """
        self._db = db

    async def init_table(self):
        """Initialize data table

        Create t_user table, skip if table already exists (idempotent operation).

        Raises:
            Error: Thrown when database operation fails
        """
        try:
            sql = f'''
            CREATE TABLE IF NOT EXISTS {self._TABLE_NAME} (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                username TEXT NOT NULL UNIQUE,
                password TEXT NOT NULL
            )
            '''
            await self._db.exec(sql)
            self._logger.info(f'succeeded to initialize {self._TABLE_NAME} table with config={self._config}')
        except Error:
            raise
        except Exception as e:
            message = f'failed to initialize {self._TABLE_NAME} table with config={self._config}'
            self._logger.error(message)
            raise Error(DbErrc.TRANSACTION_FAILED.value, message) from e

    async def insert(self, user: UserField) -> int:
        """Insert user

        Args:
            user: User field object

        Returns:
            User ID
        """
        try:
            sql = f'INSERT INTO {self._TABLE_NAME} (username, password) VALUES (?, ?)'
            await self._db.exec(sql, (user.username, user.password))
            result = await self._db.exec('SELECT last_insert_rowid()')
            id = result[0][0] if result else None
            self._logger.info(f'succeeded to insert user with username={user.username}, id={id}')
            return id
        except Error:
            raise
        except Exception as e:
            message = f'failed to insert user with username={user.username}'
            self._logger.error(message)
            raise Error(DbErrc.INSERT_FAILED.value, message) from e

    async def update_by_id(self, id: int, params: dict) -> None:
        """Update user by ID

        Args:
            id: User ID
            params: Update parameter dictionary, supported keys:
                - username: Username (optional)
                - password: Password (optional)

        Raises:
            Error: Thrown when update fails
        """
        try:
            updates = []
            sql_params = []

            if 'username' in params:
                updates.append('username = ?')
                sql_params.append(params["username"])
            if 'password' in params:
                updates.append('password = ?')
                sql_params.append(params["password"])

            if not updates:
                return

            sql_params.append(id)
            sql = f'UPDATE {self._TABLE_NAME} SET {", ".join(updates)} WHERE id = ?'
            await self._db.exec(sql, sql_params)
            self._logger.info(f'succeeded to update user with id={id}, params={params}')
        except Error:
            raise
        except Exception as e:
            message = f'failed to update user with id={id}, params={params}'
            self._logger.error(message)
            raise Error(DbErrc.UPDATE_FAILED.value, message) from e

    async def delete_by_id(self, id: int) -> None:
        """Delete user by ID

        Args:
            id: User ID

        Raises:
            Error: Thrown when deletion fails
        """
        try:
            sql = f'DELETE FROM {self._TABLE_NAME} WHERE id = ?'
            await self._db.exec(sql, (id,))
            self._logger.info(f'succeeded to delete user with id={id}')
        except Error:
            raise
        except Exception as e:
            message = f'failed to delete user with id={id}'
            self._logger.error(message)
            raise Error(DbErrc.DELETE_FAILED.value, message) from e

    async def find_by_id(self, id: int) -> UserField | None:
        """Find user by ID

        Args:
            id: User ID

        Returns:
            User field object, returns None if not found
        """
        try:
            sql = f'SELECT id, username, password FROM {self._TABLE_NAME} WHERE id = ?'
            rows = await self._db.exec(sql, (id,))
            user_dict = rows[0] if rows else None
            if user_dict is None:
                self._logger.debug(f'succeeded to find user by id with id={id}, result=None')
                return None
            result = UserField(
                username=user_dict["username"],
                password=user_dict["password"]
            )
            self._logger.debug(f'succeeded to find user by id with id={id}, result={result}')
            return result
        except Error:
            raise
        except Exception as e:
            message = f'failed to find user by id with id={id}'
            self._logger.error(message)
            raise Error(DbErrc.QUERY_FAILED.value, message) from e

    async def find_by_username(self, username: str) -> UserField | None:
        """Find user by username

        Args:
            username: Username

        Returns:
            User field object, returns None if not found
        """
        try:
            sql = f'SELECT id, username, password FROM {self._TABLE_NAME} WHERE username = ?'
            rows = await self._db.exec(sql, (username,))
            user_dict = rows[0] if rows else None
            if user_dict is None:
                self._logger.debug(f'succeeded to find user by username with username={username}, result=None')
                return None
            result = UserField(
                username=user_dict["username"],
                password=user_dict["password"]
            )
            self._logger.debug(f'succeeded to find user by username with username={username}, result={result}')
            return result
        except Error:
            raise
        except Exception as e:
            message = f'failed to find user by username with username={username}'
            self._logger.error(message)
            raise Error(DbErrc.QUERY_FAILED.value, message) from e

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
        try:
            offset = (page - 1) * page_size
            username = params.get('username')

            if username:
                sql = f'SELECT id, username, password FROM {self._TABLE_NAME} WHERE username = ? LIMIT ? OFFSET ?'
                rows = await self._db.exec(sql, (username, page_size, offset))
                count_sql = f'SELECT COUNT(*) FROM {self._TABLE_NAME} WHERE username = ?'
                count_rows = await self._db.exec(count_sql, (username,))
            else:
                sql = f'SELECT id, username, password FROM {self._TABLE_NAME} LIMIT ? OFFSET ?'
                rows = await self._db.exec(sql, (page_size, offset))
                count_sql = f'SELECT COUNT(*) FROM {self._TABLE_NAME}'
                count_rows = await self._db.exec(count_sql)

            total = count_rows[0][0] if count_rows else 0
            results = [
                UserField(username=row['username'], password=row['password'])
                for row in (rows or [])
            ]

            # Build pagination info
            total_pages = (total + page_size - 1) // page_size if total > 0 else 0
            pagination = Pagination(
                page=page,
                page_size=page_size,
                total=total,
                total_pages=total_pages,
                has_next=page < total_pages if total_pages > 0 else False,
                has_prev=page > 1 and total > 0
            )

            self._logger.debug(
                f'succeeded to find users with params={params}, page={page}, page_size={page_size}, pagination={pagination}'
            )
            return results, pagination
        except Error:
            raise
        except Exception as e:
            message = f'failed to find users with params={params}, page={page}, page_size={page_size}'
            self._logger.error(message)
            raise Error(DbErrc.QUERY_FAILED.value, message) from e
```