## Field Layer

### Module Location

`app/feature/{name}/field.py`

### Field Layer Responsibilities

1. **Data Carrier**: Defined with `@dataclass`, serves as data transfer object between modules
2. **No Business Logic**: Field only stores data, contains no business logic
3. **Type Definition**: Clearly defines type and default value for each field
4. **Serialization Support**: Provides `to_dict()` / `from_dict()` methods for dictionary conversion

### Complete UserField Template

```python
from typing import Optional
from dataclasses import dataclass


@dataclass
class UserField:
    """User field object

    Attributes:
        id: User ID, None means not saved to database
        username: Username
        password: Password
    """
    id: Optional[int] = None
    username: str = ''
    password: str = ''

    def to_dict(self):
        """Convert to dictionary

        Returns:
            Dictionary representation
        """
        return {
            "id": self.id,
            "username": self.username,
            "password": self.password
        }

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
        return cls(**user_dict)

    def __str__(self):
        """Convert to string

        Returns:
            Human-readable string representation
        """
        return f'UserField(id={self.id}, username={self.username})'
```

### Field Initialization Example

```python
from app.feature.user.field import UserField

# Create new User (not saved to database)
new_user = UserField(
    username="test_user",
    password="hashed_password"
)

# Create saved User
saved_user = UserField(
    id=1,
    username="test_user",
    password="hashed_password"
)

# Convert to dictionary
user_dict = saved_user.to_dict()

# Create object from dictionary
user = UserField.from_dict(user_dict)

# Print object string
print(saved_user)
# Output: UserField(id=1, username=test_user)
```
