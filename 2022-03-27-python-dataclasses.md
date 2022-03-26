# Python dataclass

> ...provides a decorator and functions for automatically adding generated special methods such as \_\_init\_\_() and \_\_repr\_\_() to user-defined classes. [1]

Example [2]:
```python
from dataclasses import dataclass, field

@dataclass
class Person:
    name: str
    address: str
    active: bool = True  # Default to True
    # If default value is not primitive, a simple = [] would not work
    # Function "field" would make a new list for each instance
    email_addresses: list[str] = field(default_factory=list)
    
    # field(init=False) prevents variables from being set manually
    # No need for __init__, that is done automatically by dataclass decorator
    # No need for custom __str__, that is done automatically as well
    
    def __post_init__(self) -> None:
        # Called after __init__
        pass
    
person = Person(name="John", address="123 Main St")
print(person)
```

[1] https://docs.python.org/3/library/dataclasses.html

[2] https://www.youtube.com/watch?v=CvQ7e6yUtnw
