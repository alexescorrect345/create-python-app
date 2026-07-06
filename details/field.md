## Field Layer

### Module Location

`app/feature/{name}/field.py`

### Field Layer Responsibilities

1. **Data Carrier**: Defined with `@dataclass`, serves as data transfer object between modules
2. **No Business Logic**: Field only stores data, contains no business logic
3. **Type Definition**: Clearly defines type and default value for each field
4. **Serialization Support**: Provides `to_dict()` / `from_dict()` methods for dictionary conversion

### Complete Field Template

```python
from typing import Any, Optional
from dataclasses import dataclass

@dataclass
class RoleField:
    """Role field object

    Attributes:
        id: Role ID
        name: Role name
    """
    id: Optional[int] = None
    name: str = ''

    def to_dict(self) -> dict[str, Any]:
        """Convert to dictionary

        Returns:
            Dictionary representation
        """
        return {
            "id": self.id,
            "name": self.name
        }

    @classmethod
    def from_dict(cls, role_dict: dict) -> Optional['RoleField']:
        """Create object from dictionary

        Args:
            role_dict: Dictionary data

        Returns:
            RoleField object
        """
        if role_dict is None:
            return None
        return cls(**role_dict)

    def __str__(self):
        """Convert to string

        Returns:
            Human-readable string representation
        """
        return f'RoleField(id={self.id}, name={self.name})'


@dataclass
class UserField:
    """User field object

    Attributes:
        id: User ID, None means not saved to database
        role_id: Role ID
        username: Username
        password: Password
        role: Role object (optional, for FULL query mode)
    """
    id: Optional[int] = None
    role_id: int = 0
    username: str = ''
    password: str = ''
    role: Optional[RoleField] = None

    def to_dict(self) -> dict[str, Any]:
        """Convert to dictionary

        Returns:
            Dictionary representation, converts role to dict when present
        """
        result = {
            "id": self.id,
            "role_id": self.role_id,
            "username": self.username,
            "password": self.password
        }
        if self.role:
            result["role"] = self.role.to_dict()
        return result

    @classmethod
    def from_dict(cls, user_dict: dict) -> Optional['UserField']:
        """Create object from dictionary

        Args:
            user_dict: Dictionary data

        Returns:
            UserField object
        """
        if user_dict is None:
            return None
        if "role" in user_dict and user_dict["role"] is not None:
            user_dict["role"] = RoleField.from_dict(user_dict["role"])
        return cls(**user_dict)

    def __str__(self):
        """Convert to string

        Returns:
            Human-readable string representation
        """
        return f'UserField(id={self.id}, role_id={self.role_id}, username={self.username}, password={self.password}, role={self.role})'
```
