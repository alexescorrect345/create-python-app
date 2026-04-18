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
from typing import Any, Optional

from app.common import Error, Pagination
from app.db import DB
from app.db.common import Errc as DbErrc
from app.feature.user.common import Errc as UserErrc, FieldType
from app.feature.user.field import UserField, RoleField

class UserDao:
    """User Data Access Object"""

    _logger = logging.getLogger(__name__)
    _TABLE_NAME = 't_user'

    def __init__(self, config: dict):
        """Initialize

        Args:
            config: Configuration dictionary
        """
        self._config = config
        self._db: Optional[DB] = None

    def set_db(self, db: DB) -> None:
        """Set database instance

        Args:
            db: Database instance
        """
        self._db = db

    async def init_table(self) -> None:
        """Initialize data table

        Create t_user table, skip if table already exists (idempotent operation).

        Raises:
            Error: Thrown when database operation fails
        """
        if self._db is None:
            message = f'missing db, failed to initialize {self._TABLE_NAME} table'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_DB.value, message)

        create_table_script = f'''
        CREATE TABLE IF NOT EXISTS {self._TABLE_NAME} (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            username TEXT NOT NULL UNIQUE,
            password TEXT NOT NULL
        );
        '''

        init_seq_script = f'''
        INSERT OR REPLACE INTO sqlite_sequence (name, seq)
        VALUES ('{self._TABLE_NAME}', 1000000);
        '''

        index_username_script = f'CREATE INDEX IF NOT EXISTS idx_user_username ON {self._TABLE_NAME}(username);'

        await self._db.exec(script=create_table_script)
        await self._db.exec(script=init_seq_script)
        await self._db.exec(script=index_username_script)
        self._logger.info(f'succeeded to initialize {self._TABLE_NAME} table with config={self._config}')

    async def insert(self, user: UserField) -> int:
        """Insert user

        Args:
            user: User field object

        Returns:
            User ID
        """
        if self._db is None:
            message = f'missing db, failed to insert user with user_field={user}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_DB.value, message)

        insert_script = f'INSERT INTO {self._TABLE_NAME} (username, password) VALUES (?, ?)'
        await self._db.exec(insert_script, (user.username, user.password))
        query_script = 'SELECT last_insert_rowid()'
        rows = await self._db.exec(query_script)
        if not rows or rows[0][0] is None:
            message = f'failed to find inserted user id with user_field={user}'
            self._logger.error(message)
            raise Error(UserErrc.FAILED_TO_FIND_INSERTED_ID.value, message)
        user_id = int(rows[0][0])
        self._logger.info(f'succeeded to insert user with user_id={user_id}, user_field={user}')
        return user_id

    async def update_by_id(self, id: int, params: dict[str, Any]) -> None:
        """Update user by ID

        Args:
            id: User ID
            params: Update parameter dictionary, supported keys:
                - username: Username (optional)
                - password: Password (optional)

        Raises:
            Error: Thrown when update fails
        """
        if self._db is None:
            message = f'missing db, failed to update user with id={id}, params={params}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_DB.value, message)

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

    async def delete_by_id(self, id: int) -> None:
        """Delete user by ID

        Args:
            id: User ID

        Raises:
            Error: Thrown when deletion fails
        """
        if self._db is None:
            message = f'missing db, failed to delete user with id={id}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_DB.value, message)

        user_field = await self.find_by_id(id)
        if user_field is None:
            message = f'missing user, failed to delete user with id={id}'
            self._logger.error(message)
            raise Error(UserErrc.MISSING_USER.value, message)

        sql = f'DELETE FROM {self._TABLE_NAME} WHERE id = ?'
        await self._db.exec(sql, (id,))
        self._logger.info(f'succeeded to delete user, user_field={user_field}')

    async def find_by_id(self, id: int, field_type: FieldType = FieldType.SIMPLE) -> Optional[UserField]:
        """Find user by ID

        Args:
            id: User ID
            field_type: Query type (FieldType.SIMPLE or FieldType.FULL)

        Returns:
            User field object, returns None if not found
        """
        if self._db is None:
            message = f'missing db, failed to find user by id with id={id}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_DB.value, message)

        if field_type == FieldType.FULL:
            script = f'''
            SELECT u.id, u.username, u.password, r.id, r.name
            FROM {self._TABLE_NAME} u
            LEFT JOIN t_role r ON u.role_id = r.id
            WHERE u.id = ?
            '''
        else:
            script = f'''
            SELECT id, username, password
            FROM {self._TABLE_NAME}
            WHERE id = ?
            '''

        rows = await self._db.exec(script, (id,))
        if not rows:
            self._logger.debug(f'succeeded to find user by id with id={id}, result=None')
            return None
        user_field = self._map_to_field(rows[0], field_type)
        self._logger.debug(f'succeeded to find user by id, user_field={user_field}')
        return user_field

    async def find_by_username(self, username: str, field_type: FieldType = FieldType.SIMPLE) -> Optional[UserField]:
        """Find user by username

        Args:
            username: Username
            field_type: Query type (FieldType.SIMPLE or FieldType.FULL)

        Returns:
            User field object, returns None if not found
        """
        if self._db is None:
            message = f'missing db, failed to find user by username with username={username}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_DB.value, message)

        if field_type == FieldType.FULL:
            script = f'''
            SELECT u.id, u.username, u.password, r.id, r.name
            FROM {self._TABLE_NAME} u
            LEFT JOIN t_role r ON u.role_id = r.id
            WHERE u.username = ?
            '''
        else:
            script = f'''
            SELECT id, username, password
            FROM {self._TABLE_NAME}
            WHERE username = ?
            '''

        rows = await self._db.exec(script, (username,))
        if not rows:
            self._logger.debug(f'succeeded to find user by username with username={username}, result=None')
            return None
        user_field = self._map_to_field(rows[0], field_type)
        self._logger.debug(f'succeeded to find user by username, user_field={user_field}')
        return user_field

    async def find(
        self,
        params: dict[str, Any],
        page: int = 1,
        page_size: int = sys.maxsize,
        field_type: FieldType = FieldType.SIMPLE
    ) -> tuple[list[UserField], Pagination]:
        """Find users

        Args:
            params: Query parameter dictionary, supported keys:
                - username: Username (optional)
            page: Page number (starting from 1)
            page_size: Number of items per page
            field_type: Query type (FieldType.SIMPLE or FieldType.FULL)

        Returns:
            User field object list with pagination info
        """
        if self._db is None:
            message = f'missing db, failed to find users with params={params}, page={page}, page_size={page_size}, field_type={field_type}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_DB.value, message)

        offset = (page - 1) * page_size
        username = params.get('username')

        where_clause = 'WHERE username = ?' if username else ''

        if field_type == FieldType.FULL:
            full_where = 'WHERE u.username = ?' if username else ''
            sql = f'''
            SELECT u.id, u.username, u.password, r.id, r.name
            FROM {self._TABLE_NAME} u
            LEFT JOIN t_role r ON u.role_id = r.id
            {full_where}
            ORDER BY u.id DESC
            LIMIT ? OFFSET ?
            '''
            count_sql = f'SELECT COUNT(*) FROM {self._TABLE_NAME} u {full_where}'
        else:
            sql = f'''
            SELECT id, username, password
            FROM {self._TABLE_NAME}
            {where_clause}
            ORDER BY id DESC
            LIMIT ? OFFSET ?
            '''
            count_sql = f'SELECT COUNT(*) FROM {self._TABLE_NAME} {where_clause}'

        if username:
            rows = await self._db.exec(sql, (username, page_size, offset))
            count_rows = await self._db.exec(count_sql, (username,))
        else:
            rows = await self._db.exec(sql, (page_size, offset))
            count_rows = await self._db.exec(count_sql)

        total = count_rows[0][0] if count_rows else 0
        total_pages = (total + page_size - 1) // page_size if total > 0 else 0
        pagination = Pagination(
            page=page,
            page_size=page_size,
            total=total,
            total_pages=total_pages,
            has_next=page < total_pages if total_pages > 0 else False,
            has_prev=page > 1 and total > 0
        )
        user_list = [self._map_to_field(row, field_type) for row in (rows if rows else [])]
        return user_list, pagination

    def _map_to_field(self, result: tuple[Any, ...], field_type: FieldType = FieldType.SIMPLE) -> UserField:
        if field_type == FieldType.FULL:
            return UserField(
                id=result[0],
                username=result[1],
                password=result[2],
                role=RoleField(
                    id=result[3],
                    name=result[4]
                )
            )
        else:
            return UserField(
                id=result[0],
                username=result[1],
                password=result[2]
            )
```

