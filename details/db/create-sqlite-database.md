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

from app.common import Errc, Error
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
                - Missing db_path config: Errc.MISSING_CONFIG
                - Invalid db_path type: Errc.INVALID_TYPE
                - Connection failed: DbErrc.CONNECT_FAILED
        """
        if "db_path" not in self._config:
            message = f"failed to validate config, missing required 'db_path' field, config keys: {list(self._config.keys())}"
            self._logger.error(message)
            raise Error(Errc.MISSING_CONFIG.value, message)

        db_path = self._config["db_path"]
        if not isinstance(db_path, (str, Path)):
            message = f'failed to validate config, db_path must be string or Path type, actual: {type(db_path).__name__}'
            self._logger.error(message)
            raise Error(Errc.INVALID_TYPE.value, message)

        db_path = Path(db_path)
        check_same_thread = self._config.get("check_same_thread", True)
        timeout_s = self._config.get("timeout_s", 5.0)
        isolation_level = self._config.get("isolation_level", None)

        try:
            if str(db_path) != ":memory:":
                if db_path.parent and str(db_path.parent) != ".":
                    db_path.parent.mkdir(parents=True, exist_ok=True)
                    self._logger.info(f'created directory for database: {db_path.parent}')

            self._connection = await aiosqlite.connect(
                database=str(db_path),
                check_same_thread=check_same_thread,
                timeout=timeout_s,
                isolation_level=isolation_level
            )

            db_type = "memory" if str(db_path) == ":memory:" else "file"
            self._logger.info(f'succeeded to connect to sqlite database, type: {db_type}, path: {db_path}')

        except Error:
            raise
        except Exception as e:
            self._connection = None
            message = f'failed to connect to sqlite database, db_path: {db_path}'
            self._logger.error(message)
            raise Error(DbErrc.CONNECT_FAILED.value, message) from e

    async def close(self) -> None:
        """Async close SQLite database connection

        Actions:
        1. Commit all uncommitted transactions
        2. Close database connection
        3. Release file handle (for file database)
        4. Clean up connection object

        Raises:
            Error: Close failed with DbErrc.DISCONNECT_FAILED
        """
        if self._connection is not None:
            try:
                await self._connection.commit()
                self._logger.info(f'committed pending transactions before closing, db_path: {self._config.get("db_path", "unknown")}')

                await self._connection.close()
                self._logger.info(f'succeeded to close sqlite database connection, db_path: {self._config.get("db_path", "unknown")}')

            except Exception as e:
                message = f'failed to close sqlite database connection, db_path: {self._config.get("db_path", "unknown")}'
                self._logger.error(message)
                raise Error(DbErrc.DISCONNECT_FAILED.value, message) from e

            finally:
                self._connection = None
        else:
            self._logger.info(f'connection already closed or never established, db_path: {self._config.get("db_path", "unknown")}')

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
                - Execution failed: DbErrc.EXECUTION_FAILED
                - Not connected: DbErrc.NOT_CONNECTED
                - Invalid params: Errc.INVALID_TYPE or DbErrc.MISSING_SCRIPT
        """
        if not isinstance(script, str):
            message = f'failed to validate script parameter, expected str type, actual: {type(script).__name__}'
            self._logger.error(message)
            raise Error(Errc.INVALID_TYPE.value, message)

        if not script.strip():
            message = f'failed to execute script, script is empty, db_path: {self._config.get("db_path", "unknown")}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_SCRIPT.value, message)

        if params is not None and not isinstance(params, tuple):
            message = f'failed to validate params parameter, expected tuple or None, actual: {type(params).__name__}'
            self._logger.error(message)
            raise Error(Errc.INVALID_TYPE.value, message)

        if not self.is_connected or self._connection is None:
            message = f'attempted to execute script without database connection, db_path: {self._config.get("db_path", "unknown")}'
            self._logger.error(message)
            raise Error(DbErrc.NOT_CONNECTED.value, message)

        cursor: Optional[aiosqlite.Cursor] = None

        try:
            cursor = await self._connection.cursor()

            self._logger.debug(f'executing script: {script}')
            if params:
                self._logger.debug(f'with params: {params}')

            if params is None:
                await cursor.execute(script)
            else:
                await cursor.execute(script, params)

            script_upper = script.strip().upper()
            is_query = (
                script_upper.startswith("SELECT") or
                script_upper.startswith("PRAGMA") or
                "RETURNING" in script_upper
            )

            if is_query:
                results = await cursor.fetchall()
                result_count = len(results) if results else 0
                self._logger.info(f'query executed successfully, returned {result_count} rows')
                return results if results else []
            else:
                if self._config.get("isolation_level") is not None:
                    await self._connection.commit()
                    self._logger.info(f'transaction committed, db_path: {self._config.get("db_path", "unknown")}')

                self._logger.info(f'script executed successfully (non-query), db_path: {self._config.get("db_path", "unknown")}')
                return None

        except Error:
            raise
        except Exception as e:
            if self._config.get("isolation_level") is not None:
                try:
                    await self._connection.rollback()
                except Exception:
                    self._logger.error(f'failed to rollback transaction, db_path: {self._config.get("db_path", "unknown")}')
                    self._logger.exception(e)

            message = f'failed to execute script, db_path: {self._config.get("db_path", "unknown")}, script: {script.strip()}'
            self._logger.error(message)
            raise Error(DbErrc.EXECUTION_FAILED.value, message) from e

        finally:
            if cursor is not None:
                await cursor.close()
                self._logger.debug(f'cursor closed, db_path: {self._config.get("db_path", "unknown")}')

    async def batch_exec(
        self,
        scripts: List[str],
        params_list: Optional[List[Optional[Tuple[Any, ...]]]] = None
    ) -> Optional[List[List[Tuple[Any, ...]]]]:
        """Async batch execute multiple scripts

        Args:
            scripts: Script string array, each element is a script to execute
            params_list: Parameter tuple array, corresponds to scripts array, each element can be None, default empty array

        Returns:
            Query result list (result list for each query) or None

        Raises:
            Error: Script execution failed with DbErrc.EXECUTION_FAILED or Errc.INVALID_TYPE
        """
        if not isinstance(scripts, list):
            message = f'failed to validate scripts parameter, expected list type, actual: {type(scripts).__name__}'
            self._logger.error(message)
            raise Error(Errc.INVALID_TYPE.value, message)

        if len(scripts) == 0:
            message = f'failed to validate scripts, list is empty, db_path: {self._config.get("db_path", "unknown")}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_SCRIPT.value, message)

        if params_list is None:
            params_list = []

        if not isinstance(params_list, list):
            message = f'failed to validate params_list parameter, expected list or None, actual: {type(params_list).__name__}'
            self._logger.error(message)
            raise Error(Errc.INVALID_TYPE.value, message)

        if len(params_list) != 0 and len(params_list) != len(scripts):
            message = f'failed to validate params_list, length {len(params_list)} must match scripts list length {len(scripts)}'
            self._logger.error(message)
            raise Error(Errc.INVALID_TYPE.value, message)

        for i, script in enumerate(scripts):
            if not isinstance(script, str):
                message = f'failed to validate scripts[{i}], expected str type, actual: {type(script).__name__}'
                self._logger.error(message)
                raise Error(Errc.INVALID_TYPE.value, message)
            if not script.strip():
                message = f'failed to validate scripts[{i}], script is empty, db_path: {self._config.get("db_path", "unknown")}'
                self._logger.error(message)
                raise Error(DbErrc.MISSING_SCRIPT.value, message)

        if len(params_list) > 0:
            for i, param in enumerate(params_list):
                if param is not None and not isinstance(param, tuple):
                    message = f'failed to validate params_list[{i}], expected tuple or None, actual: {type(param).__name__}'
                    self._logger.error(message)
                    raise Error(Errc.INVALID_TYPE.value, message)

        if not self.is_connected or self._connection is None:
            message = f'attempted to execute scripts without database connection, db_path: {self._config.get("db_path", "unknown")}'
            self._logger.error(message)
            raise Error(DbErrc.NOT_CONNECTED.value, message)

        cursor: Optional[aiosqlite.Cursor] = None
        result_list: List[List[Tuple[Any, ...]]] = []
        has_query = False

        try:
            cursor = await self._connection.cursor()

            script_count = len(scripts)
            self._logger.info(f'batch executing {script_count} scripts')

            for i, script in enumerate(scripts):
                self._logger.debug(f'executing script {i+1}/{script_count}: {script.strip()}')

                param = params_list[i] if len(params_list) > 0 else None
                if param is not None:
                    self._logger.debug(f'with param: {param}')

                if param is None:
                    await cursor.execute(script)
                else:
                    await cursor.execute(script, param)

                script_upper = script.strip().upper()
                is_query = (
                    script_upper.startswith("SELECT") or
                    script_upper.startswith("PRAGMA") or
                    "RETURNING" in script_upper
                )

                if is_query:
                    has_query = True
                    results = await cursor.fetchall()
                    result_list.append(results if results else [])
                else:
                    result_list.append([])

            if self._config.get("isolation_level") is not None:
                await self._connection.commit()
                self._logger.info(f'transaction committed, db_path: {self._config.get("db_path", "unknown")}')

            self._logger.info(f'batch executed successfully, {script_count} scripts completed')

            if has_query:
                return result_list
            return None

        except Error:
            raise
        except Exception as e:
            if self._config.get("isolation_level") is not None:
                try:
                    await self._connection.rollback()
                except Exception:
                    self._logger.error(f'failed to rollback transaction, db_path: {self._config.get("db_path", "unknown")}')
                    self._logger.exception(e)

            message = f'failed to batch execute scripts, db_path: {self._config.get("db_path", "unknown")}, script_count: {script_count}'
            self._logger.error(message)
            raise Error(DbErrc.EXECUTION_FAILED.value, message) from e

        finally:
            if cursor is not None:
                await cursor.close()
                self._logger.debug(f'cursor closed, db_path: {self._config.get("db_path", "unknown")}')
```
