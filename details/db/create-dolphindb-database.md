## Prerequisites

- Python 3.11+
- Project Error exception class
- DB abstract base class
- Errc error code enum
- DolphinDB Python API (`dolphindb`)

## Implementation

```python
import asyncio
import logging
from typing import Any, Dict, List, Optional, Tuple

import dolphindb as ddb

from app.common import Error
from app.db import DB
from app.db.common import Errc as DbErrc


class DolphinDB(DB):
    """DolphinDB database implementation (async version)

    Async DolphinDB database operations based on dolphindb Session. The
    synchronous dolphindb Python API is wrapped with asyncio.to_thread.

    Configuration:
        - host (str): DolphinDB server host address
        - port (int): DolphinDB server port number
        - userid (str): Username for login
        - password (str): Password for login
        - reconnect_count (int, optional): Number of reconnection attempts, auto reconnection is enabled when greater than 0, default 0
        - read_timeout_s (int, optional): Read timeout in seconds, default None
        - write_timeout_s (int, optional): Write timeout in seconds, default None

    Attributes:
        - is_connected: Read-only connection state (determined by whether _connection is None)
    """

    _logger = logging.getLogger(__name__)

    @classmethod
    def to_literal(cls, value: Any) -> str:
        """Convert a Python value to a DolphinDB literal string.

        Args:
            value: Value to convert (None/bool/int/float/str)

        Returns:
            DolphinDB literal string

        Raises:
            Error: Unsupported value type with DbErrc.INVALID_VALUE_TYPE
        """
        if value is None:
            # DolphinDB NULL literal
            return 'NULL'

        # bool must be checked before int: in Python bool is a subclass of int
        # (isinstance(True, int) is True), otherwise True/False would become '1'/'0'.
        if isinstance(value, bool):
            return 'true' if value else 'false'

        if isinstance(value, int):
            return str(value)

        if isinstance(value, float):
            # repr keeps full precision (e.g. 0.1) better than str
            return repr(value)

        if isinstance(value, str):
            # wrap in double quotes; escape backslash, double quote and control chars
            escaped_value = (
                value
                .replace('\\', '\\\\')
                .replace('"', '\\"')
                .replace('\n', '\\n')
                .replace('\r', '\\r')
                .replace('\t', '\\t')
            )
            return f'"{escaped_value}"'

        message = f'unsupported dolphindb literal type with value_type={type(value).__name__}, value={value}'
        cls._logger.error(message)
        raise Error(DbErrc.INVALID_VALUE_TYPE.value, message)

    def __init__(self, config: Dict[str, Any]) -> None:
        """Initialize DolphinDB database connection

        Args:
            config: Configuration dict
        """
        super().__init__(config)

    async def connect(self) -> None:
        """Async connect to DolphinDB database

        Behavior:
        1. Validate host, port, userid, and password configuration
        2. Create a DolphinDB session
        3. Connect with reconnect and timeout settings

        Raises:
            Error:
                - Missing host config: DbErrc.MISSING_HOST
                - Missing port config: DbErrc.MISSING_PORT
                - Missing userid config: DbErrc.MISSING_USERID
                - Missing password config: DbErrc.MISSING_PASSWORD
                - Connection failed: DbErrc.FAILED_TO_CONNECT
        """
        if "host" not in self._config:
            message = f'missing host with config={self._config}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_HOST.value, message)

        if "port" not in self._config:
            message = f'missing port with config={self._config}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_PORT.value, message)

        if "userid" not in self._config:
            message = f'missing userid with config={self._config}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_USERID.value, message)

        if "password" not in self._config:
            message = f'missing password with config={self._config}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_PASSWORD.value, message)

        host = self._config["host"]
        port = self._config["port"]
        userid = self._config["userid"]
        password = self._config["password"]
        reconnect_count = self._config.get("reconnect_count", 0)
        read_timeout_s = self._config.get("read_timeout_s", None)
        write_timeout_s = self._config.get("write_timeout_s", None)

        try:
            self._connection = ddb.Session()

            await asyncio.to_thread(
                self._connection.connect,
                host=host,
                port=port,
                userid=userid,
                password=password,
                reconnect=reconnect_count > 0,
                tryReconnectNums=reconnect_count if reconnect_count > 0 else None,
                readTimeout=read_timeout_s,
                writeTimeout=write_timeout_s,
            )

            self._logger.info(f'succeeded to connect to dolphindb database with host={host}, port={port}, userid={userid}')

        except Exception as e:
            self._connection = None
            message = f'failed to connect to dolphindb database with host={host}, port={port}, userid={userid}'
            self._logger.error(message)
            raise Error(DbErrc.FAILED_TO_CONNECT.value, message) from e

    async def close(self) -> None:
        """Async close DolphinDB database connection

        Actions:
        1. Close database connection
        2. Clean up connection object

        Raises:
            Error: Close failed with DbErrc.FAILED_TO_DISCONNECT
        """
        if self.is_connected():
            try:
                await asyncio.to_thread(self._connection.close)
                self._logger.info(f'succeeded to close dolphindb database connection with host={self._config["host"]}, port={self._config["port"]}, userid={self._config["userid"]}')

            except Exception as e:
                message = f'failed to close dolphindb database connection with host={self._config["host"]}, port={self._config["port"]}, userid={self._config["userid"]}'
                self._logger.error(message)
                raise Error(DbErrc.FAILED_TO_DISCONNECT.value, message) from e

            finally:
                self._connection = None
        else:
            self._logger.info(f'succeeded to close dolphindb database connection with host={self._config["host"]}, port={self._config["port"]}, userid={self._config["userid"]}')

    async def exec(
        self,
        script: str,
        params: Optional[Tuple[Any, ...]] = None
    ) -> Any:
        """Async execute script

        Supported operations:
        - Query/function/script execution: Returns DolphinDB execution result
        - Scripts without a return value: Returns None

        Args:
            script: Script string or function name to execute
            params: Positional arguments passed to Session.run, can be None

        Returns:
            Execution result, or None if no return value

        Raises:
            Error:
                - Empty script: DbErrc.MISSING_SCRIPT
                - Not connected: DbErrc.NOT_CONNECTED
                - Execution failed: DbErrc.FAILED_TO_EXEC
        """
        if not script.strip():
            message = f'missing script with host={self._config["host"]}, port={self._config["port"]}, userid={self._config["userid"]}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_SCRIPT.value, message)

        if not self.is_connected():
            message = f'did not connect to database with host={self._config["host"]}, port={self._config["port"]}, userid={self._config["userid"]}'
            self._logger.error(message)
            raise Error(DbErrc.NOT_CONNECTED.value, message)

        try:
            result = await self._exec_once(script, params)
            return result

        except Exception as e:
            message = f'failed to execute script with host={self._config["host"]}, port={self._config["port"]}, userid={self._config["userid"]}, script={script.strip()}, params={params}'
            self._logger.error(message)
            raise Error(DbErrc.FAILED_TO_EXEC.value, message) from e

    async def batch_exec(
        self,
        scripts: List[str],
        params_list: List[Optional[Tuple[Any, ...]]] = []
    ) -> List[Any]:
        """Async batch execute multiple scripts

        Supported operations:
        - Query/function/script execution: Returns one result per script, in order
        - Scripts without a return value: Returns None for that script

        Args:
            scripts: Script string list, each element is a DolphinDB script to execute
            params_list: Positional argument tuple list corresponding to scripts, each element can be None, default empty list

        Returns:
            Execution result list in script order (None preserved for non-returning scripts)

        Raises:
            Error:
                - Empty scripts: DbErrc.MISSING_SCRIPT
                - Not connected: DbErrc.NOT_CONNECTED
                - Execution failed: DbErrc.FAILED_TO_EXEC
        """
        if not scripts:
            message = f'missing scripts with host={self._config["host"]}, port={self._config["port"]}, userid={self._config["userid"]}'
            self._logger.error(message)
            raise Error(DbErrc.MISSING_SCRIPT.value, message)

        if not self.is_connected():
            message = f'did not connect to database with host={self._config["host"]}, port={self._config["port"]}, userid={self._config["userid"]}'
            self._logger.error(message)
            raise Error(DbErrc.NOT_CONNECTED.value, message)

        result_list: List[Any] = []

        try:
            script_count = len(scripts)
            self._logger.debug(f'batch executing {script_count} scripts with host={self._config["host"]}, port={self._config["port"]}, userid={self._config["userid"]}')

            for i, script in enumerate(scripts):
                params = params_list[i] if i < len(params_list) else None
                result = await self._exec_once(script, params)
                result_list.append(result)

            self._logger.debug(f'succeeded to batch execute {script_count} scripts with host={self._config["host"]}, port={self._config["port"]}, userid={self._config["userid"]}')

            return result_list

        except Exception as e:
            message = f'failed to batch execute scripts with host={self._config["host"]}, port={self._config["port"]}, userid={self._config["userid"]}, scripts={scripts}, params_list={params_list}'
            self._logger.error(message)
            raise Error(DbErrc.FAILED_TO_EXEC.value, message) from e

    async def _exec_once(
        self,
        script: str,
        params: Optional[Tuple[Any, ...]] = None
    ) -> Any:
        """Execute a single DolphinDB script on the current session (no connection/empty validation, no transaction management).

        Args:
            script: DolphinDB script or function name to execute
            params: Positional arguments tuple passed to Session.run, can be None

        Returns:
            DolphinDB execution result, or None if no return value
        """
        self._logger.debug(f'executing script={script.strip()}, params={params}')

        if params is None:
            result = await asyncio.to_thread(self._connection.run, script)
        else:
            result = await asyncio.to_thread(self._connection.run, script, *params)

        self._logger.debug(f'succeeded to execute script with script={script.strip()}, params={params}')
        return result
```
