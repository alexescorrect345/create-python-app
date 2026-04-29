## Task Layer

### Module Location

`app/feature/user/task.py`

### Task Layer Responsibilities

1. **Background task execution**: Scheduled execution, loop execution, one-time execution
2. **Call Service**: Execute business logic through the Service layer
3. **Exception handling**: Catch exceptions during task execution, log errors, do not affect the task loop
4. **Lifecycle management**: Start task, stop task, query task status

### Rules

#### Service Null Check

Before executing any Service-related operations, must check if the Service has been initialized. If the Service is not found, log the error and raise a business exception.

```python
# ✅ Correct: Check Service before executing
async def _run_once(self):
    if self._user_service is None:
        message = f'failed to find user_service in UserTask'
        self._logger.error(message)
        raise Error(CommonErrc.MISSING_SERVICE.value, message)
    
    users = await self._user_service.find()

# ❌ Wrong: Call without checking
async def _run_once(self):
    users = await self._user_service.find()  # May be None
```

#### Loop Interval

The `_run_once` method must include an `await asyncio.sleep()` call, otherwise the task will enter an infinite loop and continuously consume CPU resources.

```python
# ✅ Correct: sleep in finally block, ensuring interval after each execution
async def _run_once(self):
    try:
        # Execute task logic
        pass
    except Exception as e:
        message = f'failed to execute task once in UserTask'
        logger.error(message)
        logger.exception(e)
    finally:
        await asyncio.sleep(60)  # Must have interval

# ❌ Wrong: No sleep, will cause infinite loop
async def _run_once(self):
    try:
        # Execute task logic
        pass
    except Exception as e:
        message = f'failed to execute task once in UserTask'
        logger.error(message)
        logger.exception(e)
    # Missing asyncio.sleep, task will immediately start next loop iteration
```

**Notes**:
- It is recommended to place `asyncio.sleep()` in the `finally` block to ensure proper interval even when exceptions occur
- The interval time should be read from configuration for easy environment adjustment

### Dependency Injection Pattern

Task uses **Setter Injection** pattern for dependency injection:

1. **Constructor injection for required dependencies**:
   - `__init__(config)` - Inject configuration dictionary

2. **Setter method injection for optional dependencies**:
   - `set_user_service(service)` - Inject user service
   - Supports chaining: `task.set_xxx_service(...).set_yyy_service(...)`

3. **Lazy initialization**:
   - Task can be created first, dependencies injected later
   - Provides more flexible object lifecycle management

### Task Initialization Example

```python
import asyncio

from app.feature.user.task import UserTask

# Initialize in main.py (optional)
# If no background task is needed, initialization can be skipped
task = UserTask(config=config)
task.set_user_service(service=user_service)
# Use create_task to start asynchronously, without blocking the startup flow
asyncio.create_task(task.run_in_loop())

# Clean up task when application shuts down
await task.close()  # Sets self._running = False, allowing the task loop to exit
```

### Complete Task Template

```python
import asyncio
import logging

from app.common import Error, Errc as CommonErrc
from app.task import Task
from app.feature.user import UserService


class UserTask(Task):
    """User background task"""
    _logger = logging.getLogger(__name__)

    def __init__(self, config: dict):
        """Initialize

        Args:
            config: Configuration dictionary
        """
        super().__init__(config=config)
        self._user_service = None

    def set_user_service(self, service: UserService) -> None:
        """Set user service

        Args:
            service: User service
        """
        self._user_service = service

    async def _run_once(self) -> None:
        """Execute task once (example: clean up expired users)"""
        # Service null check
        if self._user_service is None:
            message = f'failed to find user_service in UserTask'
            self._logger.error(message)
            raise Error(CommonErrc.MISSING_SERVICE.value, message)

        try:
            # Get all users
            users = await self._user_service.find()
            self._logger.info(f'succeeded to check {len(users)} users in UserTask')

            # Cleanup logic (example)
            for user in users:
                # Example: check user status
                self._logger.debug(f'checking user with username={user.username} in UserTask')

            self._logger.info(f'succeeded to execute task once in UserTask with checked_count={len(users)}')
        except Error as e:
            message = f'failed to execute task once in UserTask with code={e.code}, message={e.message}'
            self._logger.error(message)
            self._logger.exception(e)
        except Exception as e:
            message = f'failed to execute task once in UserTask with config={self._config}'
            self._logger.error(message)
            self._logger.exception(e)
        finally:
            # Sleep regardless of success or failure
            interval_s = self._config["task_interval_s"]
            await asyncio.sleep(interval_s)
```
