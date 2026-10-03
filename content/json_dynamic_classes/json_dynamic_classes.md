[Next Article](../logging/logging.md) \| [Next Article](../descriptors/descriptors.md)

## JSON Parsing through DOT notation by Creating JSON models dynamically

In the previous two articles, we explored how to parse JSON data into Python objects using 
object-oriented design. We created Python classes manually to represent the structure of the
JSON data, making it possible to work with the data using familiar dot notation.

In this article, we will take this approach a step further. Instead of manually defining Python 
classes for every JSON structure, we will create the Python objects dynamically from the JSON
data itself. we will build a simple approach to dynamically transform JSON data into Python 
objects that can be accessed using dot notation.

This allows us to work with different JSON structures without having to create a corresponding
set of Python classes beforehand, while still providing a clean and intuitive dot-notation
interface for accessing nested data.

Consider the below class `JsonObject`
```python
from json import dumps

class JsonObject:
    """Represent a JSON object as a Python object."""
    def __init__(self, info):
        self._info = info

    def __getattr__(self, name):
        try:
            return self._info[name]
        except KeyError:
            raise AttributeError(f"{self.__class__.__name__} has no attribute {name!r}") from None

    def __setattr__(self, name, value):
        """Set an attribute (_info) or update the underlying JSON data."""
        if name == "_info":
            super().__setattr__(name, value)
        else:
            self._info[name] = value

    def to_dict(self):
        """Convert a JsonObject to dictionary."""
        out_dict = {}
        for key, value in self._info.items():
            if isinstance(value, JsonObject):
                out_dict[key] = value.to_dict()
            elif isinstance(value, list):
                out_dict[key] = [item.to_dict() if isinstance(item, JsonObject) else item for item in value]
            else:
                out_dict[key] = value
        return out_dict

    def to_json(self):
        """Convert JsonObject to a JSON string."""
        return dumps(self.to_dict())
```
This above class is the core building block of our dynamic JSON model. 
It wraps a Python dictionary and exposes its data through **dot notation**, allowing 
JSON properties to be accessed like regular Python attributes.

The `JsonObject` class provides three functionalities,
* **Provide dot-notation access** It allows JSON properties to be accessed as Python 
attributes, such as json.name or json.address.city, instead of using dictionary keys.
* **Allow data to be updated using dot notation** Assigning a value such 
as `json.name = "Steve"` updates the underlying JSON data rather than creating a separate
Python attribute.
* **Convert the object back to JSON-compatible data** to_dict() converts the `JsonObject` 
back into a Python dictionary, including nested objects and lists, while `to_json()` 
serializes that dictionary into a `JSON` string.

```python
from json import loads, load

class JsonPath:
    def __init__(self):
        """Initialize an empty JSON parser."""
        self._root_json_object = None

    def __getattr__(self, name):
        """Delegate attribute access to the json object."""
        return getattr(self._root_json_object, name)

    def __setattr__(self, name, value):
        if name == "_root_json_object":
            super().__setattr__(name, value)
        else:
            setattr(self._root_json_object, name, value)

    def _wrap_dict(self, data):
        """Wrap a dictionary in a JsonObject."""
        return JsonObject(data)

    def _process_list(self, data: list):
        """Recursively process items in a list."""
        return [self._process_object(item) for item in data]

    def _process_dict(self, data: dict):
        """Recursively process values in a dictionary."""
        _data = {key: self._process_object(value) for key, value in data.items()}
        return self._wrap_dict(_data)

    def _process_object(self, value):
        """Process a JSON object based on its Python type."""
        if isinstance(value, dict):
            return self._process_dict(value)
        elif isinstance(value, list):
            return self._process_list(value)
        return value

    def _build_root_object(self, py_object):
        """Build the root object from the deserialized JSON."""
        return self._process_object(py_object)

    @classmethod
    def _from_py_object(cls, py_object):
        if isinstance(py_object, (dict, list)) and len(py_object) == 0:
            raise ValueError(f"Cannot create a JSON model from an empty object/response: {py_object}")
        obj = cls()
        obj._root_json_object = obj._build_root_object(py_object)
        return obj

    @classmethod
    def from_json_string(cls, json_string):
        """Create a JsonPath instance from a JSON string."""
        return cls._from_py_object(loads(json_string))

    @classmethod
    def from_json_file(cls, filename: str):
        """Create a JsonPath instance from a JSON file."""
        path = Path(filename)
        if not path.is_file():
            raise FileNotFoundError(f"File not found {filename}")
        with open(path, mode="r", encoding="utf-8") as json_file:
            return cls._from_py_object(load(json_file))
```