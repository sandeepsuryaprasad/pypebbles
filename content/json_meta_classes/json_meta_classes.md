## Creating JSON models dynamically

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

```python
from json import loads, load

class JsonPath:
    def __init__(self):
        """Initialize an empty JSON parser."""
        self._json_object = None

    def __getattr__(self, name):
        """Delegate attribute access to the json object."""
        return getattr(self._json_object, name)

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
        obj = cls()
        obj._json_object = obj._build_root_object(py_object)
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