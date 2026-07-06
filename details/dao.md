## DAO Layer

### Module Location

`app/feature/user/dao.py`

### DAO Layer Responsibilities

1. **Only responsible for database persistence operations**: Execute database scripts or database helper calls, return raw persistence results
2. **No business logic handling**: No data validation, no business error throwing
3. **Throw database errors**: Only throw DB errors
4. **Parameter validation**: Not validated at DAO layer (handled by Service layer)
5. **Database-specific implementation is allowed**: DAO methods can use different access patterns based on the selected database (for example, SQL scripts for SQLite, DFS/table helper calls for DolphinDB)

### Rules

#### Method Ordering

The order of methods in a DAO class should follow this sequence:

1. **init_table**: Table initialization
2. **insert / batch_insert**: Insert and related methods (e.g., insert, batch_insert)
3. **upsert / upsert_***: Upsert and related methods (DolphinDB-specific and optional, e.g., upsert_by_id)
4. **update / update_***: Update and related methods (e.g., update_by_id, update)
5. **delete / delete_***: Delete and related methods (e.g., delete_by_id, delete)
6. **find / find_***: Find and related methods (e.g., find_by_id, find_by_username, find)

This order ensures a logical flow from table setup → data manipulation → data retrieval.

Skip methods that are not needed by the selected database. For example, `upsert` usually only exists in DolphinDB-based DAOs.

#### Database Script String Conventions

- Use **single quotes** for SQL statements or other database query script strings
- When the statement itself contains single quotes, use **double quotes**
- For database scripts that need line breaks, use **three single quotes** (`'''`)
- Query/script fragments used in concatenation (such as column names, condition fragments, etc.) must also follow the rules above

```python
# ✅ Correct: Use single quotes for database scripts
script = f'SELECT id, username, password FROM {self._TABLE_NAME} WHERE id = ?'
script = f'INSERT INTO {self._TABLE_NAME} (username, password) VALUES (?, ?)'
updates.append('username = ?')

# ✅ Correct: Use three single quotes for multi-line database scripts
script = f'''
CREATE TABLE IF NOT EXISTS {self._TABLE_NAME} (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT NOT NULL UNIQUE,
    password TEXT NOT NULL
)
'''

# ✅ Correct: Use double quotes when the script itself contains single quotes
script = f"SELECT * FROM {self._TABLE_NAME} WHERE username = '{username}'"

# ❌ Wrong: Use double quotes for a normal script string
script = f"SELECT id, username, password FROM {self._TABLE_NAME} WHERE id = ?"
# ❌ Wrong: Use three double quotes for a multi-line script
script = f"""
CREATE TABLE IF NOT EXISTS {self._TABLE_NAME} (
    id INTEGER PRIMARY KEY AUTOINCREMENT
)
"""
```

The examples above use SQL-style scripts for illustration. When using `DolphinDB`, keep the same string formatting rules, but follow DolphinDB script syntax and database helper APIs instead of SQLite-specific SQL syntax.

#### Variable Naming for Database Fields

- If a variable represents a field in a database record, the variable naming should match the database field name (when no conflict exists)
- For example: If the primary key field of table `t_user` is `id`, use `id` instead of `user_id` as the variable name

```python
# ✅ Correct: Use variable name matching the database field
script = f'SELECT id, username, password FROM {self._TABLE_NAME} WHERE id = ?'
rows = await self._db.exec(script=script, params=(id,))

# ❌ Wrong: Use user_id instead of id
script = f'SELECT id, username, password FROM {self._TABLE_NAME} WHERE id = ?'
rows = await self._db.exec(script=script, params=(user_id,))  # Should use id
```

The naming rule above applies to all supported databases. The code snippet is SQL-style only; when using `DolphinDB`, keep the same field names but use DolphinDB query scripts or DB helpers instead of SQLite-style parameter placeholders.

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

### Complete UserDao Template (SQLite Example)

> ⚠️ The following example is a `SqliteDB`-specific DAO template. When using `DolphinDB`, create a database-specific DAO implementation with DolphinDB table initialization, write helpers, query scripts, result mapping, and ID handling.

```python
import sys
import logging
from typing import Any, Optional

from app.common import Error, Pagination
from app.db import DB
from app.db.common import Errc as DbErrc
from app.feature.user.common import FieldType
from app.feature.user.field import RoleField, UserField

class UserDao:
    """User Data Access Object"""

    _logger = logging.getLogger(__name__)
    _TABLE_NAME = 't_user'
    _INIT_SEQ = 1000000

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
        This template assumes there is also a `t_role` table with `id` and `name`.

        Raises:
            Error: Thrown when database operation fails
        """
        if self._db is None:
            message = f'missing db with {self._TABLE_NAME}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_DB.value, message)

        create_table_script = f'''
        CREATE TABLE IF NOT EXISTS {self._TABLE_NAME} (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            username TEXT NOT NULL UNIQUE,
            password TEXT NOT NULL,
            role_id INTEGER
        );
        '''

        init_seq_script = f'''
        INSERT OR REPLACE INTO sqlite_sequence (name, seq)
        VALUES ('{self._TABLE_NAME}', {self._INIT_SEQ});
        '''

        index_username_script = f'CREATE INDEX IF NOT EXISTS idx_user_username ON {self._TABLE_NAME}(username);'

        await self._db.batch_exec(
            scripts=[create_table_script, init_seq_script, index_username_script]
        )
        self._logger.info(f'succeeded to initialize {self._TABLE_NAME} table')

    async def insert(self, user_field: UserField) -> Any:
        """Insert user

        Args:
            user_field: User field object

        Returns:
            User ID
        """
        if self._db is None:
            message = f'missing db with user_field={user_field}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_DB.value, message)

        insert_script = f'INSERT INTO {self._TABLE_NAME} (username, password, role_id) VALUES (?, ?, ?)'
        query_script = 'SELECT last_insert_rowid()'
        rows_lists = await self._db.batch_exec(
            scripts=[insert_script, query_script],
            params_list=[(user_field.username, user_field.password, user_field.role_id), None]
        )
        rows = rows_lists[1]
        if not rows or rows[0][0] is None:
            message = f'failed to find inserted user id with user_field={user_field}'
            self._logger.error(message)
            raise Error(DbErrc.FAILED_TO_FIND_INSERTED_ID.value, message)
        id = int(rows[0][0])
        self._logger.debug(f'succeeded to insert user with id={id}, user_field={user_field}')
        return id

    async def update_by_id(self, id: Any, params: dict[str, Any]) -> None:
        """Update user by ID

        Args:
            id: User ID
            params: Update parameter dictionary, supported keys:
                - username: Username (optional)
                - password: Password (optional)
                - role_id: Role ID (optional)

        Raises:
            Error: Thrown when update fails
        """
        if self._db is None:
            message = f'missing db with id={id}, params={params}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_DB.value, message)

        updates = []
        script_params = []

        if "username" in params:
            updates.append('username = ?')
            script_params.append(params["username"])
        if "password" in params:
            updates.append('password = ?')
            script_params.append(params["password"])
        if "role_id" in params:
            updates.append('role_id = ?')
            script_params.append(params["role_id"])

        if not updates:
            return

        script_params.append(id)
        script = f'UPDATE {self._TABLE_NAME} SET {", ".join(updates)} WHERE id = ?'
        await self._db.exec(script=script, params=tuple(script_params))
        self._logger.debug(f'succeeded to update user with id={id}, params={params}')

    async def delete_by_id(self, id: Any) -> None:
        """Delete user by ID

        Args:
            id: User ID

        Raises:
            Error: Thrown when deletion fails
        """
        if self._db is None:
            message = f'missing db with id={id}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_DB.value, message)

        script = f'DELETE FROM {self._TABLE_NAME} WHERE id = ?'
        await self._db.exec(script=script, params=(id,))
        self._logger.debug(f'succeeded to delete user with id={id}')

    async def find_by_id(self, id: Any, field_type: FieldType = FieldType.SIMPLE) -> Optional[UserField]:
        """Find user by ID

        Args:
            id: User ID
            field_type: Query type (FieldType.SIMPLE or FieldType.FULL)

        Returns:
            User field object, returns None if not found
        """
        if self._db is None:
            message = f'missing db with id={id}, field_type={field_type}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_DB.value, message)

        if field_type == FieldType.FULL:
            script = f'''
            SELECT u.id, u.username, u.password, u.role_id, r.id, r.name
            FROM {self._TABLE_NAME} u
            LEFT JOIN t_role r ON u.role_id = r.id
            WHERE u.id = ?
            '''
        else:
            script = f'''
            SELECT id, username, password, role_id
            FROM {self._TABLE_NAME}
            WHERE id = ?
            '''

        rows = await self._db.exec(script=script, params=(id,))
        if not rows:
            self._logger.debug(f'succeeded to find user by id with id={id}, result=None')
            return None
        user_field = self._map_to_field(rows[0], field_type)
        self._logger.debug(f'succeeded to find user by id with id={id}, user_field={user_field}')
        return user_field

    async def find(
        self,
        params: Optional[dict[str, Any]] = None,
        orderby: Optional[list[tuple[str, str]]] = None,
        field_type: FieldType = FieldType.SIMPLE,
        page: int = 1,
        page_size: int = sys.maxsize,
    ) -> tuple[list[UserField], Pagination]:
        """Find users

        Args:
            params: Query parameter dictionary, supported keys:
                - username: Username (optional)
                - password: Password (optional)
                - role_id: Role ID (optional)
            orderby: Order spec list, each item is a (field_name, direction) tuple;
                direction must be 'asc' or 'desc'; order follows list order,
                defaults to [('id', 'desc')] when None or empty
            field_type: Query type (FieldType.SIMPLE or FieldType.FULL)
            page: Page number (starting from 1)
            page_size: Number of items per page

        Returns:
            User field object list with pagination info
        """
        params = params or {}
        if self._db is None:
            message = f'missing db with params={params}, field_type={field_type}, page={page}, page_size={page_size}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_DB.value, message)

        offset = (page - 1) * page_size
        conditions = []
        values: list[Any] = []

        if params.get("username") is not None:
            conditions.append('u.username = ?')
            values.append(params["username"])

        if params.get("password") is not None:
            conditions.append('u.password = ?')
            values.append(params["password"])

        if params.get("role_id") is not None:
            conditions.append('u.role_id = ?')
            values.append(params["role_id"])

        where_clause = ('WHERE ' + ' AND '.join(conditions)) if conditions else ''

        order_parts = []
        for field, direction in (orderby or [('id', 'desc')]):
            if direction not in ('asc', 'desc'):
                message = f'invalid order direction with field={field}, direction={direction}'
                self._logger.error(message)
                raise Error(DbErrc.INVALID_ORDERBY.value, message)
            order_parts.append(f'u.{field} {direction.upper()}')

        order_clause = 'ORDER BY ' + ', '.join(order_parts)

        if field_type == FieldType.FULL:
            script = f'''
            SELECT u.id, u.username, u.password, u.role_id, r.id, r.name
            FROM {self._TABLE_NAME} u
            LEFT JOIN t_role r ON u.role_id = r.id
            {where_clause}
            {order_clause}
            LIMIT ? OFFSET ?
            '''
        else:
            script = f'''
            SELECT id, username, password, role_id
            FROM {self._TABLE_NAME} u
            {where_clause}
            {order_clause}
            LIMIT ? OFFSET ?
            '''

        count_script = f'SELECT COUNT(*) FROM {self._TABLE_NAME} u {where_clause}'

        values.extend([page_size, offset])
        # exclude pagination params (page_size, offset) for count query
        count_values = values[:-2]

        rows_list = await self._db.batch_exec(
            scripts=[script, count_script],
            params_list=[tuple(values), tuple(count_values) if count_values else None]
        )
        rows = rows_list[0]
        count_rows = rows_list[1]

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
        items = [self._map_to_field(result, field_type) for result in (rows if rows else [])]
        self._logger.debug(f'succeeded to find users with params={params}, field_type={field_type}, page={page}, page_size={page_size}, total={total}')
        return items, pagination

    def _map_to_field(self, result: tuple[Any, ...], field_type: FieldType = FieldType.SIMPLE) -> UserField:
        if field_type == FieldType.FULL:
            return UserField(
                id=result[0],
                username=result[1],
                password=result[2],
                role_id=result[3],
                role=RoleField(
                    id=result[4],
                    name=result[5]
                )
            )
        else:
            return UserField(
                id=result[0],
                username=result[1],
                password=result[2],
                role_id=result[3]
            )
```

### Complete UserDao Template (DolphinDB Example)

> ⚠️ This example uses DolphinDB scripts executed through `exec` and `batch_exec`. Because DolphinDB has no SQLite-style auto-increment, `insert` generates IDs internally via `max(id) + 1` (with `_INIT_SEQ` as the floor) and returns the generated ID — matching the SQLite DAO contract. Upsert uses `upsert!`, and query operations load DFS tables with DolphinDB scripts. For `append!` and `upsert!`, keep the input table columns aligned with the target DFS table schema, including the actual `id` type.

```python
import sys
import logging
from typing import Any, Optional, Union

import pandas as pd

from app.common import Error, Pagination
from app.db import DB
from app.db.common import Errc as DbErrc
from app.db.DolphinDB import DolphinDB
from app.feature.user.common import FieldType
from app.feature.user.field import RoleField, UserField

class UserDao:
    """User Data Access Object"""

    _logger = logging.getLogger(__name__)
    _TABLE_NAME = 't_user'
    _INIT_SEQ = 1000000

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
        """Initialize DFS table

        Create t_user table when it does not exist in the configured DolphinDB database.
        This template assumes there is also a `t_role` table with `id` and `name`.

        Raises:
            Error: Thrown when database operation fails
        """
        if self._db is None:
            message = f'missing db with {self._TABLE_NAME}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_DB.value, message)

        create_table_script = f'''
        if (!existsTable("dfs://{self._config["db_path"]}", "{self._TABLE_NAME}")) {{
            create table "dfs://{self._config["db_path"]}"."{self._TABLE_NAME}"(
                id INT,
                username STRING,
                password STRING,
                role_id INT
            )
            partitioned by id,
            primaryKey=`id
        }}
        '''

        await self._db.exec(create_table_script)
        self._logger.info(f'succeeded to initialize {self._TABLE_NAME} table in dfs://{self._config["db_path"]}')

    async def insert(self, user_field: UserField) -> Any:
        """Insert user

        Note:
            DolphinDB has no SQLite-style auto-increment. This method generates
            the ID internally via `max(id) + 1` (with `_INIT_SEQ` as the floor
            for empty tables) so that the DAO contract matches the SQLite DAO:
            the caller passes a UserField without an ID and receives the
            generated ID back. It is a thin wrapper over `batch_insert`.

        Args:
            user_field: User field object

        Returns:
            Generated user ID

        Raises:
            Error: Thrown when database operation fails
        """
        ids = await self.batch_insert([user_field])
        return ids[0]

    async def batch_insert(self, user_list: list[UserField]) -> list[Any]:
        """Insert a batch of users

        Note:
            DolphinDB has no SQLite-style auto-increment. This method generates
            IDs internally via `max(id) + 1, max(id) + 2, ...` (with `_INIT_SEQ`
            as the floor for empty tables) so that the DAO contract matches the
            SQLite DAO: the caller passes UserFields without IDs and receives
            the generated IDs back. The single-row `insert` is a thin wrapper
            around this method.

            NOTE: For high-concurrency scenarios that require strict
            serializability, replace this max+1 approach with a dedicated
            sequence table to avoid race conditions.

        Args:
            user_list: List of user field objects

        Returns:
            List of generated user IDs, in the same order as the input

        Raises:
            Error: Thrown when database operation fails
        """
        if not user_list:
            self._logger.debug(f'skip dolphindb batch_insert because user_list is empty with table={self._TABLE_NAME}')
            return []

        if self._db is None:
            message = f'missing db with user_list={user_list}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_DB.value, message)

        # Generate sequential IDs starting from max(id) + 1 (or _INIT_SEQ + 1 for empty tables).
        max_id_script = f'select max(id) as max_id from loadTable("dfs://{self._config["db_path"]}", "{self._TABLE_NAME}")'
        max_rows = await self._db.exec(max_id_script)
        val = max_rows.iloc[0]["max_id"] if (max_rows is not None and not max_rows.empty) else None
        base_id = self._INIT_SEQ if (val is None or pd.isna(val)) else int(val)

        ids = [base_id + i for i in range(1, len(user_list) + 1)]
        for item, id in zip(user_list, ids):
            item.id = id

        await self._db.exec(
            script=f'append!{{loadTable("dfs://{self._config["db_path"]}", "{self._TABLE_NAME}")}}',
            params=(pd.DataFrame([item.to_dict() for item in user_list]),),
        )
        self._logger.debug(f'succeeded to insert user with ids={ids}, user_count={len(user_list)}')
        return ids

    async def upsert(self, user_field: Union[UserField, list[UserField]]) -> None:
        """Upsert user

        Note:
            This template uses DolphinDB `upsert!` with `id` as the key column.
            Do not use this method for pure inserts, because its performance is
            significantly worse than `insert`. Accepts either a single
            UserField or a list of UserField.

        Args:
            user_field: User field object or list of user field objects
        """
        if self._db is None:
            message = f'missing db with user_field={user_field}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_DB.value, message)

        user_list = user_field if isinstance(user_field, list) else [user_field]
        if not user_list:
            self._logger.debug(f'skip dolphindb upsert because user_list is empty with table={self._TABLE_NAME}')
            return None

        await self._db.exec(
            script=f'upsert!{{loadTable("dfs://{self._config["db_path"]}", "{self._TABLE_NAME}")}}',
            # ignoreNull=False means NULL values in newData will also overwrite existing values.
            params=(pd.DataFrame([item.to_dict() for item in user_list]), False, 'id'),
        )
        self._logger.debug(f'succeeded to upsert user with user_count={len(user_list)}')
        return None

    async def update_by_id(self, id: Any, params: dict[str, Any]) -> None:
        """Update user by ID

        Args:
            id: User ID
            params: Update parameter dictionary, supported keys:
                - username: Username (optional)
                - password: Password (optional)
                - role_id: Role ID (optional)

        Raises:
            Error: Thrown when update fails
        """
        if self._db is None:
            message = f'missing db with id={id}, params={params}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_DB.value, message)

        updates = []

        if "username" in params:
            updates.append(f'username = {DolphinDB.to_literal(params["username"])}')
        if "password" in params:
            updates.append(f'password = {DolphinDB.to_literal(params["password"])}')
        if "role_id" in params:
            updates.append(f'role_id = {DolphinDB.to_literal(params["role_id"])}')

        if not updates:
            return

        script = f'''
        update loadTable("dfs://{self._config["db_path"]}", "{self._TABLE_NAME}")
        set {", ".join(updates)}
        where id = {DolphinDB.to_literal(id)}
        '''

        await self._db.exec(script)
        self._logger.debug(f'succeeded to update user with id={id}, params={params}')

    async def delete_by_id(self, id: Any) -> None:
        """Delete user by ID

        Args:
            id: User ID

        Raises:
            Error: Thrown when deletion fails
        """
        if self._db is None:
            message = f'missing db with id={id}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_DB.value, message)

        script = f'''
        delete from loadTable("dfs://{self._config["db_path"]}", "{self._TABLE_NAME}")
        where id = {DolphinDB.to_literal(id)}
        '''

        await self._db.exec(script)
        self._logger.debug(f'succeeded to delete user with id={id}')

    async def find_by_id(self, id: Any, field_type: FieldType = FieldType.SIMPLE) -> Optional[UserField]:
        """Find user by ID

        Args:
            id: User ID
            field_type: Query type (FieldType.SIMPLE or FieldType.FULL)

        Returns:
            User field object, returns None if not found
        """
        if self._db is None:
            message = f'missing db with id={id}, field_type={field_type}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_DB.value, message)

        if field_type == FieldType.FULL:
            script = f'''
            t_user = select id, username, password, role_id from loadTable("dfs://{self._config["db_path"]}", "{self._TABLE_NAME}") where id = {DolphinDB.to_literal(id)}
            t_role = loadTable("dfs://{self._config["db_path"]}", "t_role")
            select t_user.id, t_user.username, t_user.password, t_user.role_id, t_role.id as t_role_id, t_role.name as role_name from lj(t_user, t_role, `role_id, `id)
            '''
        else:
            script = f'''
            select id, username, password, role_id from loadTable("dfs://{self._config["db_path"]}", "{self._TABLE_NAME}") where id = {DolphinDB.to_literal(id)}
            '''

        rows = await self._db.exec(script)
        if rows is None or rows.empty:
            self._logger.debug(f'succeeded to find user by id with id={id}, result=None')
            return None

        user_field = self._map_to_field(rows.iloc[0], field_type)
        self._logger.debug(f'succeeded to find user by id with id={id}, user_field={user_field}')
        return user_field

    async def find(
        self,
        params: Optional[dict[str, Any]] = None,
        orderby: Optional[list[tuple[str, str]]] = None,
        field_type: FieldType = FieldType.SIMPLE,
        page: int = 1,
        page_size: int = sys.maxsize,
    ) -> tuple[list[UserField], Pagination]:
        """Find users

        Args:
            params: Query parameter dictionary, supported keys:
                - username: Username (optional)
                - password: Password (optional)
                - role_id: Role ID (optional)
            orderby: Order spec list, each item is a (field_name, direction) tuple;
                direction must be 'asc' or 'desc'; order follows list order,
                defaults to [('id', 'desc')] when None or empty
            field_type: Query type (FieldType.SIMPLE or FieldType.FULL)
            page: Page number (starting from 1)
            page_size: Number of items per page

        Returns:
            User field object list with pagination info
        """
        params = params or {}
        if self._db is None:
            message = f'missing db with params={params}, field_type={field_type}, page={page}, page_size={page_size}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_DB.value, message)

        offset = (page - 1) * page_size
        conditions = []

        if params.get("username") is not None:
            conditions.append(f'username = {DolphinDB.to_literal(params["username"])}')
        if params.get("password") is not None:
            conditions.append(f'password = {DolphinDB.to_literal(params["password"])}')
        if params.get("role_id") is not None:
            conditions.append(f'role_id = {DolphinDB.to_literal(params["role_id"])}')

        where_clause = ('where ' + ' and '.join(conditions)) if conditions else ''

        order_parts = []
        for field, direction in (orderby or [('id', 'desc')]):
            if direction not in ('asc', 'desc'):
                message = f'invalid order direction with field={field}, direction={direction}'
                self._logger.error(message)
                raise Error(DbErrc.INVALID_ORDERBY.value, message)
            order_parts.append(f'{field} {direction}')

        order_clause = 'order by ' + ', '.join(order_parts)

        if field_type == FieldType.FULL:
            query_script = f'''
            t_user = select id, username, password, role_id from loadTable("dfs://{self._config["db_path"]}", "{self._TABLE_NAME}") {where_clause} {order_clause} limit {offset}, {page_size}
            t_role = loadTable("dfs://{self._config["db_path"]}", "t_role")
            select t_user.id, t_user.username, t_user.password, t_user.role_id, t_role.id as t_role_id, t_role.name as role_name from lj(t_user, t_role, `role_id, `id)
            '''
        else:
            query_script = f'''
            select id, username, password, role_id from loadTable("dfs://{self._config["db_path"]}", "{self._TABLE_NAME}") {where_clause} {order_clause} limit {offset}, {page_size}
            '''

        count_script = f'''
        select count(*) as total
        from loadTable("dfs://{self._config["db_path"]}", "{self._TABLE_NAME}")
        {where_clause}
        '''

        result_list = await self._db.batch_exec(
            scripts=[
                query_script,
                count_script,
            ]
        )
        user_df = result_list[0] if result_list[0] is not None else pd.DataFrame()
        count_df = result_list[1] if result_list[1] is not None else pd.DataFrame()

        total = int(count_df.iloc[0]["total"]) if not count_df.empty else 0
        total_pages = (total + page_size - 1) // page_size if total > 0 else 0
        pagination = Pagination(
            page=page,
            page_size=page_size,
            total=total,
            total_pages=total_pages,
            has_next=page < total_pages if total_pages > 0 else False,
            has_prev=page > 1 and total > 0,
        )
        items = [self._map_to_field(row, field_type) for _, row in user_df.iterrows()]
        self._logger.debug(f'succeeded to find users with params={params}, field_type={field_type}, page={page}, page_size={page_size}, total={total}')
        return items, pagination

    def _map_to_field(self, row: pd.Series, field_type: FieldType) -> UserField:
        if field_type == FieldType.FULL:
            return UserField(
                id=row["id"],
                username=str(row["username"]),
                password=str(row["password"]),
                role_id=row["role_id"],
                role=RoleField(
                    id=row["t_role_id"],
                    name=str(row["role_name"]),
                ),
            )
        return UserField(
            id=row["id"],
            username=str(row["username"]),
            password=str(row["password"]),
            role_id=row["role_id"],
        )
```
