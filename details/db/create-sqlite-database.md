## Prerequisites

- Python 3.11+
- Project Error exception class
- DB abstract base class
- Errc error code enum
- Standard library sqlite3 module

## Implementation

```python
import logging
from typing import Any, Dict, List, Optional, Tuple
from pathlib import Path

import aiosqlite
import aiofiles.os

from app.common import Errc as CommonErrc, Error
from app.db import DB
from app.db.common import Errc as DbErrc


class SqliteDB(DB):
    """SQLite database implementation (async version)

    Async SQLite database operations based on aiosqlite library.

    Configuration:
        - db_path (str|Path): Database file path, supports relative and absolute paths
        - check_same_thread (bool, optional): Whether to allow cross-thread connection sharing, default True
        - timeout_s (float, optional): Connection timeout in seconds, default 5.0
        - isolation_level (str|None, optional): Isolation level, default None (auto-commit)

    Attributes:
        - is_connected: Read-only connection state (determined by whether _connection is None)
    """

    _logger = logging.getLogger(__name__)

    def __init__(self, config: Dict[str, Any]) -> None:
        """Initialize SQLite database connection

        Args:
            config: Configuration dict
        """
        super().__init__(config)

    async def connect(self) -> None:
        """Async connect to SQLite database

        Behavior:
        1. Validate db_path configuration
        2. For in-memory database (:memory:), create memory connection directly
        3. For file database:
           - If file exists, load existing database
           - If file doesn't exist, create new database file
           - Auto-create parent directory if path specified and directory doesn't exist

        Raises:
            Error:
                - Missing db_path config: DbErrc.MISSING_DB_PATH
                - Failed to create directory: DbErrc.FAILED_TO_MK_DB_DIR
                - Connection failed: DbErrc.FAILED_TO_CONNECT
        """
        if "db_path" not in self._config:
            message = f'missing db_path with config={self._config}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_DB_PATH.value, message)

        db_path = self._config["db_path"]
        check_same_thread = self._config.get("check_same_thread", True)
        timeout_s = self._config.get("timeout_s", 5.0)
        isolation_level = self._config.get("isolation_level", None)

        try:
            if db_path != ':memory:':
                db_path_obj = Path(db_path)
                if db_path_obj.parent and str(db_path_obj.parent) != ".":
                    await aiofiles.os.makedirs(str(db_path_obj.parent), exist_ok=True)
                    self._logger.info(f'succeeded to create directory for database with db_path={db_path}')
        except Error:
            raise
        except Exception as e:
            message = f'failed to create directory for database with db_path={db_path}'
            self._logger.error(message)
            raise Error(DbErrc.FAILED_TO_MK_DB_DIR.value, message) from e

        try:
            self._connection = await aiosqlite.connect(
                database=db_path,
                check_same_thread=check_same_thread,
                timeout=timeout_s,
                isolation_level=isolation_level
            )

            self._logger.info(f'succeeded to connect to sqlite database with db_path={db_path}')

        except Error:
            raise
        except Exception as e:
            self._connection = None
            message = f'failed to connect to sqlite database with db_path={db_path}'
            self._logger.error(message)
            raise Error(DbErrc.FAILED_TO_CONNECT.value, message) from e

    async def close(self) -> None:
        """Async close SQLite database connection

        Actions:
        1. Commit all uncommitted transactions
        2. Close database connection
        3. Release file handle (for file database)
        4. Clean up connection object

        Raises:
            Error: Close failed with DbErrc.FAILED_TO_DISCONNECT
        """
        if self._connection is not None:
            try:
                await self._connection.commit()
                self._logger.info(f'succeeded to commit pending transactions before closing with db_path={self._config.get("db_path", "unknown")}')

                await self._connection.close()
                self._logger.info(f'succeeded to close sqlite database connection, db_path: {self._config.get("db_path", 'unknown')}')

            except Exception as e:
                message = f'failed to close sqlite database connection, db_path: {self._config.get("db_path", 'unknown')}'
                self._logger.error(message)
                raise Error(DbErrc.FAILED_TO_DISCONNECT.value, message) from e

            finally:
                self._connection = None
        else:
            self._logger.info(f'succeeded to close sqlite database connection with db_path={self._config.get("db_path", "unknown")}')

    async def exec(
        self,
        script: str,
        params: Optional[Tuple[Any, ...]] = None
    ) -> Optional[List[Tuple[Any, ...]]]:
        """Async execute script

        Supported operations:
        - SELECT/PRAGMA: Returns query result list
        - INSERT/UPDATE/DELETE: Returns None, or result list if contains RETURNING clause
        - CREATE/DROP: Returns None

        Args:
            script: Script string to execute, can be SQL statement
            params: SQL parameter tuple for parameterized queries, can be None

        Returns:
            Query result list, or None if no return value

        Raises:
            Error:
                - Empty script: DbErrc.MISSING_SCRIPT
                - Not connected: DbErrc.NOT_CONNECTED
                - Execution failed: DbErrc.FAILED_TO_EXEC
        """
        if not script.strip():
            message = f'missing script with db_path={self._config.get("db_path", "unknown")}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_SCRIPT.value, message)

        if self._connection is None:
            message = f'did not connect to database with db_path={self._config.get("db_path", "unknown")}'
            self._logger.error(message)
            raise Error(DbErrc.NOT_CONNECTED.value, message)

        cursor: Optional[aiosqlite.Cursor] = None

        try:
            cursor = await self._connection.cursor()

            self._logger.debug(f'executing script={script}, params={params}')

            if params is None:
                await cursor.execute(script)
            else:
                await cursor.execute(script, params)

            script_upper = script.strip().upper()
            is_query = (
                script_upper.startswith('SELECT') or
                script_upper.startswith('PRAGMA') or
                'RETURNING' in script_upper
            )

            if is_query:
                rows = await cursor.fetchall()
                row_count = len(rows) if rows else 0
                self._logger.debug(f'query executed successfully, returned {row_count} rows')
                return rows if rows else []
            else:
                if self._config.get("isolation_level") is not None:
                    await self._connection.commit()
                    self._logger.debug(f'transaction committed, db_path: {self._config.get("db_path", "unknown")}')

                self._logger.debug(f'succeeded to execute non-query script with db_path={self._config.get("db_path", "unknown")}')
                return None

        except Error:
            raise
        except Exception as e:
            if self._config.get("isolation_level") is not None:
                try:
                    await self._connection.rollback()
                except Exception as e1:
                    self._logger.error(f'failed to rollback transaction with db_path={self._config.get("db_path", "unknown")}')
                    self._logger.exception(e1)

            message = f'failed to execute script with db_path={self._config.get("db_path", "unknown")}, script={script.strip()}, params={params}'
            self._logger.error(message)
            raise Error(DbErrc.FAILED_TO_EXEC.value, message) from e

        finally:
            if cursor is not None:
                try:
                    await cursor.close()
                    self._logger.debug(f'succeeded to close cursor with db_path={self._config.get("db_path", "unknown")}')
                except Exception as e1:
                    self._logger.error(f'failed to close cursor with db_path={self._config.get("db_path", "unknown")}')
                    self._logger.exception(e1)

    async def batch_exec(
        self,
        scripts: List[str],
        params_list: List[Optional[Tuple[Any, ...]]] = []
    ) -> List[List[Tuple[Any, ...]]]:
        """Async batch execute multiple scripts

        Supported operations:
        - SELECT/PRAGMA: Returns query result list for each query
        - INSERT/UPDATE/DELETE: Returns result list if contains RETURNING clause, otherwise skipped
        - CREATE/DROP: Skipped in result list

        Args:
            scripts: Script string list, each element is a SQL statement to execute
            params_list: SQL parameter tuple list corresponding to scripts, each element can be None, default empty list

        Returns:
            Query result list (only query scripts), or empty list if no query scripts

        Raises:
            Error:
                - Empty scripts: DbErrc.MISSING_SCRIPT
                - Not connected: DbErrc.NOT_CONNECTED
                - Execution failed: DbErrc.FAILED_TO_EXEC
        """
        if not scripts:
            message = f'missing scripts with db_path={self._config.get("db_path", "unknown")}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_SCRIPT.value, message)

        if self._connection is None:
            message = f'did not connect to database with db_path={self._config.get("db_path", "unknown")}'
            self._logger.error(message)
            raise Error(DbErrc.NOT_CONNECTED.value, message)

        cursor: Optional[aiosqlite.Cursor] = None
        result_list: List[List[Tuple[Any, ...]]] = []

        try:
            cursor = await self._connection.cursor()

            script_count = len(scripts)
            self._logger.debug(f'batch executing {script_count} scripts with db_path={self._config.get("db_path", "unknown")}')

            for i, script in enumerate(scripts):
                params = params_list[i] if i < len(params_list) else None

                self._logger.debug(f'executing script {i+1}/{script_count}, script={script.strip()}, params={params}')

                if params is None:
                    await cursor.execute(script)
                else:
                    await cursor.execute(script, params)

                script_upper = script.strip().upper()
                is_query = (
                    script_upper.startswith('SELECT') or
                    script_upper.startswith('PRAGMA') or
                    'RETURNING' in script_upper
                )

                if is_query:
                    rows = await cursor.fetchall()
                    result_list.append(rows if rows else [])

            if self._config.get("isolation_level") is not None:
                await self._connection.commit()
                self._logger.debug(f'succeeded to commit transaction with db_path={self._config.get("db_path", "unknown")}')

            self._logger.debug(f'succeeded to batch execute {script_count} scripts with db_path={self._config.get("db_path", "unknown")}')

            return result_list

        except Error:
            raise
        except Exception as e:
            if self._config.get("isolation_level") is not None:
                try:
                    await self._connection.rollback()
                except Exception as e1:
                    self._logger.error(f'failed to rollback transaction with db_path={self._config.get("db_path", "unknown")}')
                    self._logger.exception(e1)

            message = f'failed to batch execute scripts with db_path={self._config.get("db_path", "unknown")}, script_count={len(scripts)}'
            self._logger.error(message)
            raise Error(DbErrc.FAILED_TO_EXEC.value, message) from e

        finally:
            if cursor is not None:
                try:
                    await cursor.close()
                    self._logger.debug(f'succeeded to close cursor with db_path={self._config.get("db_path", "unknown")}')
                except Exception as e1:
                    self._logger.error(f'failed to close cursor with db_path={self._config.get("db_path", "unknown")}')
                    self._logger.exception(e1)
```
