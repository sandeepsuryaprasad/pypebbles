### Creating JSON models dynamically

```python
class JsonMeta(type):
    """Metaclass for dynamically creating JSON object classes."""
    def __new__(cls, clsname, bases=(), clsdict=None):
        clsdict = clsdict if clsdict else {}
        clsdict["__init__"] = lambda self, info: self.__dict__.update(info)
        return super().__new__(cls, clsname, (), clsdict)
```

```python
from json import loads
from typing import Sequence, Mapping


class JsonPath:
    def __init__(self, json_string: str):
        """Initialize the JSON parser with a JSON string."""
        self.json_string = json_string
        self._py_object = self._deserialize_json
        self.root = self.build_root_object(self._py_object)

    def _create_dynamic_class(self, name):
        """Create a dynamic JSON object class."""
        return JsonMeta(name)

    @property
    def _deserialize_json(self):
        """Deserialize the JSON string into a Python object."""
        return loads(self.json_string)

    def _wrap_dict(self, data):
        """Wrap a dictionary in a dynamically created object."""
        # create a dynamic class and wrap the dictionary inside the class
        dynamic_class = self._create_dynamic_class("JsonObject")
        return dynamic_class(data)  # return object instance of the dynamic class

    def _process_list_like_object(self, list_like_object: Sequence):
        """Recursively process items in a list-like object."""
        items = []
        for item in list_like_object:
            items.append(self._process_object(item))
        return items

    def _process_dict_like_object(self, dict_like_object: Mapping):
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

    def build_root_object(self, py_object):
        """Build the root object from the deserialized JSON."""
        return self._process_object(py_object)
```