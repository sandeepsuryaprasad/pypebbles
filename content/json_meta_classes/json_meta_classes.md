### Creating JSON models dynamically

```python
class JsonObject:
    """Represent a JSON object as a Python object."""
    def __init__(self, info):
        self.__dict__.update(info)
```

```python
from json import loads

class JsonPath:
    def __init__(self, json_string: str):
        """Initialize the JSON parser with a JSON string."""
        self.json_string = json_string
        self._py_object = self._deserialize_json()
        self.root = self._build_root_object(self._py_object)

    def _deserialize_json(self):
        """Deserialize the JSON string into a Python object."""
        return loads(self.json_string)

    def _wrap_dict(self, data):
        """Wrap a dictionary in a dynamically created object."""
        return JsonObject(data)

    def _process_list_like_object(self, list_like_object: list):
        """Recursively process items in a list-like object."""
        items = []
        for item in list_like_object:
            items.append(self._process_object(item))
        return items

    def _process_dict_like_object(self, dict_like_object: dict):
        """Recursively process values in a dictionary-like object."""
        data = {}
        for key, value in dict_like_object.items():
            data[key] = self._process_object(value)
        return self._wrap_dict(data)

    def _process_object(self, value):
        """Process a JSON object based on its Python type."""
        if isinstance(value, dict):
            return self._process_dict_like_object(value)
        elif isinstance(value, list):
            return self._process_list_like_object(value)
        return value

    def _build_root_object(self, py_object):
        """Build the root object from the deserialized JSON."""
        return self._process_object(py_object)
```