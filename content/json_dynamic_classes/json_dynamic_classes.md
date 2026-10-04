[Previous Article](../descriptors/descriptors.md) \| [Next Article](../logging/logging.md)

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

### Approach

The goal is to transform a JSON response into a hierarchy of Python objects that can be
accessed using dot notation, without manually defining Python classes for each JSON
structure.

To keep the implementation simple and maintainable, the parsing process is divided into 
multiple levels of abstraction. At the lowest level, the `JsonObject` class represents 
an individual JSON object. It wraps a Python dictionary and provides dot-notation access 
to its properties. It is also responsible for converting the object back to a 
dictionary or JSON string.

The `JsonPath` class acts as the parsing layer. It takes the deserialized JSON data 
and recursively examines each value. Dictionaries are converted into `JsonObject` instances,
while lists are processed element by element. This allows nested JSON objects and 
arrays to be represented naturally as Python objects.

Finally, the public interface hides these implementation details from the user. 
The user only needs to provide a JSON string or file and can then access the resulting 
data using familiar Python dot notation.

This separation of responsibilities gives us a simple flow:

`JsonPath` - Parses and builds the object hierarchy

`JsonObject` - Wraps parsed response and provides access to individual JSON objects

By separating parsing from object representation, each part of the implementation has a
clear responsibility and can be developed and tested independently.

The `JsonPath` class is **The Parsing and Object-Building Layer** and the **entry point 
for converting JSON data into our dynamic Python object model.**

It supports two forms of input:
* A JSON string through `from_json_string()`
* A JSON file through `from_json_file()`

Once the JSON has been parsed, `JsonPath` recursively walks through dictionaries and lists. 
Every dictionary is converted into a JsonObject, while primitive values such as `strings`, 
`numbers`, `booleans`, and `None` are retained as they are.

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
    
    def __getitem__(self, index):
        """Return an item from the root JSON array."""
        return self._root_json_object[index]

    def __len__(self):
        """Return the length of root JSON array"""
        return len(self._root_json_object)

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
The most important thing to understand is that `JsonPath` does not know the details of how
a JSON object behaves. It is responsible for walking the JSON structure and building the 
hierarchy.

### Representing JSON Objects

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

The `JsonObject` class represents an individual JSON object within the dynamically built 
object hierarchy. While `JsonPath` is responsible for parsing the JSON structure and 
recursively building the hierarchy, `JsonObject` is responsible for 
**representing the individual JSON objects and providing an object-oriented interface to 
their data**.

The class has three main responsibilities:

* Provide access to JSON properties using **dot notation** 
such as `json.name` or `json.address.city`, instead of using dictionary keys
* Allow JSON properties to be modified using **dot notation/attribute assignment**. 
value such as `json.name = "Steve"` updates the underlying JSON data rather than creating 
a separate Python attribute.
* Convert the object hierarchy back into a Python dictionary or JSON string using `to_dict()`
and `to_json()`

### Final Thoughts

In the previous two articles, we created Python classes manually to represent the structure 
of JSON data. While that approach provides a clear and strongly defined model, 
it can become repetitive when working with different or frequently changing JSON structures.

In this article, we took that idea a step further by building the Python object model 
dynamically from the JSON data itself. `JsonPath` handles the parsing and recursively 
builds the object hierarchy, while `JsonObject` provides a simple object-oriented interface 
for accessing and modifying the data using dot notation.

The result is a lightweight approach that allows us to work with complex and nested JSON 
structures without having to manually create Python classes for every response.

A practical use case for this approach is working with **REST APIs that return large or 
deeply nested JSON responses**. Instead of repeatedly navigating dictionaries such  
as `response["user"]["address"]["city"]`, we can work with the response using a more  natural 
Python syntax such as `response.user.address.city`. This can make code that consumes API 
responses easier to read and maintain, particularly when the response contains multiple 
levels of nested objects and lists.

The implementation presented here intentionally focuses on the core concept and keeps 
the design simple. There are several areas that could be extended when turning this 
into a reusable open-source library, such as stronger validation, handling special
property names, additional Python object behavior, and more comprehensive test coverage.

The main takeaway is that **JSON parsing does not have to be limited to dictionaries 
and key-based access**. With a small amount of abstraction, we can dynamically transform 
JSON into a Python object hierarchy that is easier and more natural to work with.
