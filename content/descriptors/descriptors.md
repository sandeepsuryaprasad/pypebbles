[Previous Article](../json/reading_json.md) \| [Next Article](../logging/logging.md)

## Building a Declarative JSON-to-Object Mapper - A Metaprogramming Approach

This article demonstrates how Python's **descriptors** and **class decorators** can be 
used to eliminate boilerplate code and improve readability when working with complex JSON 
data. We will use a class decorator to inspect a class's declarative field mappings and
dynamically attach descriptor instances to the class. Each descriptor implements attribute
access through the **descriptor protocol**, retrieving the corresponding value from the 
underlying JSON data and, when required, converting nested dictionaries into instances of 
their mapped Python classes. The class decorator also injects an initializer to associate 
the JSON data with each object. Together, these mechanisms provide a declarative approach 
to mapping JSON structures to Python objects, enabling intuitive dot-notation access without
manually implementing repetitive attribute-handling logic.

Let us begin with a representative JSON response containing reservation and passenger
information. The original data consists of a collection of reservation records, with each 
record representing a passenger's reservation details. To keep the example focused on 
descriptors and class decorators, we will work with a single representative record from 
that collection.

`reservations.json`
```console
[
    {
      "passenger": {
        "passenger_id": "PAX-104582",
        "first_name": "John",
        "last_name": "Doe",
        "contact": {
          "email": "john.doe@example.com",
          "phone": "+1-123-555-0987",
          "alternate_phone": "+1-111-222-3333"
        },
        "address": {
          "city": "Sampleville",
          "state": "California",
          "country": "United States"
        }
      },
      "flight": {
        "flight_number": "SA1234",
        "flight_status": "SCHEDULED",
        "aircraft": {
          "registration": "N003AN",
          "manufacturer": "Boeing",
          "model": "737-800",
          "configuration": "160"
        },
        "seat": {
          "number": "12A",
          "cabin": "Business",
          "position": "Window",
          "is_exit_row": false,
          "is_extra_legroom": true
        },
        "services": [
          {
            "code": "MEAL",
            "name": "Premium Meal",
            "description": "Chicken and roasted vegetables",
            "status": "CONFIRMED"
          },
          {
            "code": "WIFI",
            "name": "Inflight Wi-Fi",
            "description": "High-speed internet access",
            "status": "CONFIRMED"
          },
          {
            "code": "LOUNGE",
            "name": "Elite Club",
            "description": "Airport lounge access",
            "status": "CONFIRMED"
          }
        ]
      }
      }
]
```

Below is the hierarchical structure of the `reservations.json` response. 

```console
Reservations (List)
└── [0] (Object)
    ├── passenger (Object)
    │   ├── passenger_id : str
    │   ├── first_name : str
    │   ├── last_name : str
    │   ├── contact (Object)
    │   │   ├── email : str
    │   │   ├── phone : str
    │   │   └── alternate_phone : str
    │   └── address (Object)
    │       ├── city : str
    │       ├── state : str
    │       └── country : str
    │
    └── flight (Object)
        ├── flight_number : str
        ├── flight_status : str
        ├── aircraft (Object)
        │   ├── registration : str
        │   ├── manufacturer : str
        │   ├── model : str
        │   └── configuration : str
        ├── seat (Object)
        │   ├── number : str
        │   ├── cabin : str
        │   ├── position : str
        │   ├── is_exit_row : bool
        │   └── is_extra_legroom : bool
        └── services (List)
            ├── [0] (Object)
            │   ├── code : str
            │   ├── name : str
            │   ├── description : str
            │   └── status : str
            ├── [1] (Object)
            │   ├── code : str
            │   ├── name : str
            │   ├── description : str
            │   └── status : str
            └── [2] (Object)
                ├── code : str
                ├── name : str
                ├── description : str
                └── status : str
```
Let us implement the `Reservations` class, which serves as the **data access layer** 
responsible for loading and deserializing reservation data from the JSON file
and making it available through a structured Python interface.

```python
from json import load
from pathlib import Path


class Reservations:
    """Provide sequence-style access to reservation records."""
    def __init__(self, file_path):
        """Initialize the reservation collection."""
        self._path = self.get_file_path(file_path)
        self._data = self._load_json_data
        self._reservations = self._get_reservations

    def get_file_path(self, file_path) -> Path:
        """Return the path to the reservations JSON file."""
        path = Path(file_path)
        if not path.exists():
            raise FileNotFoundError(f"{path} does not exist")
        return path

    @property
    def _load_json_data(self):
        """Load reservation data from the JSON file."""
        with open(self._path, "r") as json_file:
            return load(json_file)

    @property
    def _get_reservations(self):
        """Convert reservation records into Reservation objects."""
        return [Reservation(reservation) for reservation in self._data]

    def __getitem__(self, index):
        """Return the reservation at the specified index."""
        return self._reservations[index]

    def __len__(self):
        """Return the number of reservations in the collection."""
        return len(self._reservations)
```
We can now instantiate the `Reservations` class to load the reservation data and access 
individual reservation records through the resulting collection.
```python
>>> reservations = Reservations("employees.json")
>>> reservations
<__main__.Reservations object at 0x107332850>
>>> type(reservations)
<class __main__.Reservations>
```
We can now access individual reservation records using standard sequence-style indexing
```python
>>> reservations[0]
<__main__.Reservation object at 0x10734e6a0>
```
Since the JSON data contains a single reservation record, 
index `0` refers to the only available record in the collection. 
The returned value is a `Reservation` object representing that record, 
rather than the raw JSON dictionary.

For the purpose of this demonstration and to keep the article concise, 
we will implement only the `Passenger`, and `Flight` classes. 
Each JSON node is mapped to its corresponding Python attribute through the `Field` 
descriptor. This provides a sufficient foundation to demonstrate how descriptors can 
abstract the underlying JSON structure and expose both scalar values and nested 
objects through a structured, attribute-based interface.

```python
@map_fields
class Reservation:
    """Represent reservation information using descriptor-based field mappings.

    The class uses the :class decorator:`map_fields` to dynamically create `Field` descriptors 
    for the JSON nodes declared in ``_nodes``.
    Scalar fields are mapped directly to their corresponding JSON values,
    while nested objects are mapped to their respective Python classes.

    The ``_nodes`` definition acts as a declarative schema that describes
    the structure of the passenger data and its corresponding Python
    representations.

    Attributes:
        _nodes: Collection of JSON field names and their corresponding
            Python types. A value of ``None`` indicates that the JSON value
            is returned directly, while a class specifies the type used to
            represent a nested JSON object.

    Example:
        ``passenger.first_name`` returns the value of the ``first_name``
        JSON field, while ``passenger.contact.email`` provides attribute-based
        access to the nested address data.
    """

    _nodes = [
        ("passenger", Passenger),
        ("flight", Flight)
    ]
```
**Mapping:**

The `_nodes` class variable defines the top-level JSON fields and their corresponding 
Python mapping classes.

- `passenger`: Maps the `passenger` JSON object to an instance of the `Passenger` class,
providing access to passenger details, contact information, and address.
- `flight`: Maps the `flight` JSON object to an instance of the `Flight` class, 
providing access to flight details, aircraft information, seat details, 
and associated services.

Each tuple contains the name of a JSON field/node and the class responsible for 
representing its data. Unlike scalar mappings that use `None`, these entries reference 
classes because both fields contain nested JSON objects.

Now let's implement `Passenger` and `Flight` classes

```python
@map_fields
class Passenger:
    """Represent passenger information using descriptor mappings.

    The `map_fields` class decorator processes the field definitions declared
    in `_nodes` and dynamically creates `Field` descriptors for accessing
    the corresponding values in the underlying JSON data.

    Scalar fields are mapped directly, while nested JSON objects are mapped
    to their corresponding Python classes. This allows passenger information
    to be accessed through a structured, object-oriented interface.

    Attributes:
        passenger_id (str): The unique identifier assigned to the passenger.
        first_name (str): The passenger's first name.
        last_name (str): The passenger's last name.
        contact (Contact): The passenger's contact information, represented
            by a `Contact` object.
        address (Address): The passenger's address information, represented
            by an `Address` object.

    Mapping:
        `_nodes` defines the passenger fields to be mapped from the JSON
        response. Each entry is a tuple containing the JSON field name and
        its associated mapping type.

        A value of `None` indicates a direct scalar field mapping, while a
        class reference specifies the mapping class used to represent a
        nested JSON object.

    The decorator uses these definitions to establish descriptor-based
    attribute access for both scalar values and nested passenger details.
    """
    _nodes = [
        ("passenger_id", None),
        ("first_name", None),
        ("last_name", None),
        ("contact", Contact),
        ("address", Address),   # Not implemented in this article
    ]
```

```python
@map_fields
class Flight:
    """Represent flight information and its associated details.

    The class defines the mapping between JSON fields and their corresponding
    Python attributes using the `_nodes` class variable.

    Attributes:
        flight_number (str): The flight's unique flight number.
        flight_status (str): The current status of the flight, such as
            SCHEDULED, BOARDING, or CANCELLED.
        aircraft (Aircraft): The aircraft assigned to the flight, represented
            by an Aircraft object.
        seat (Seat): The passenger's seat details, represented by a Seat
            object.
        services (Services): The flight's associated services, represented
            by a Services object.

    Mapping:
        `_nodes` defines the fields to be mapped from the JSON response.
        Each tuple contains a field name and its associated node mapping.
        A value of None indicates a direct field mapping, while a class
        reference specifies the mapping to use for nested data.

    The `@map_fields` decorator processes this mapping definition to
    establish the corresponding attributes on the Flight class.
    """
    _nodes = [
        ("flight_number", None),
        ("flight_status", None),
        ("aircraft", Aircraft),     # Not implemented in this article
        ("seat", Seat),             # Not implemented in this article
        ("services", Services)
    ]
```
The `contact` node in the `Passenger` class is mapped to the `Contact` class, 
which encapsulates the passenger's contact information. Let us implement the 
`Contact` class to define the mapping for its individual fields.

```python
@map_fields
class Contact:
    """Represent a passenger's contact information using descriptor mappings.

    The `map_fields` class decorator processes the field definitions declared
    in `_nodes` and dynamically creates `Field` descriptors for accessing
    the corresponding values in the underlying JSON data.

    Each field represents a scalar contact property and is mapped directly
    without additional object conversion.

    Attributes:
        email (str): The passenger's primary email address.
        phone (str): The passenger's primary contact phone number.
        alternate_phone (str): An alternative phone number for contacting
            the passenger.

    Mapping:
        `_nodes` defines the contact fields to be mapped from the JSON
        response. Each entry is a tuple containing the JSON field name and
        its associated mapping type. A value of `None` indicates that the
        field is mapped directly as a scalar value.

    The decorator uses these definitions to provide descriptor-based access
    to the contact information through the corresponding Python attributes.
    """

    _nodes = [
        ("email", None),
        ("phone", None),
        ("alternate_phone", None),
    ]
```
Similarly, The `services` node in the `Flight` class is mapped to the `Services` class, 
which encapsulates the service information. Let us implement the `Services` class to 
define the mapping for its individual fields.
```python
class Services:
    """Represent a collection of flight services using mapped service objects.

    The class encapsulates a list of services associated with a flight.
    Each service is represented by an instance of the nested `Service` class,
    which uses the `map_fields` decorator to provide descriptor-based access
    to the service properties.

    The constructor accepts a list of service dictionaries and converts each
    dictionary into a `Service` instance. The resulting objects are stored
    internally in `_services` and can be accessed individually using the
    indexing operator.

    Attributes:
        _services (list[Services.Service]): The collection of mapped service
            objects created from the supplied service information.

    Nested Class:
        Service: Represents an individual flight service and maps its JSON
            fields to Python attributes.

            Attributes:
                code (str): The service code, such as MEAL, WIFI, or LOUNGE.
                name (str): The display name of the service.
                description (str): A description of the service.
                status (str): The current status of the service, such as
                    CONFIRMED.

    Methods:
        __init__(services_info): Convert the supplied list of service
            dictionaries into `Service` instances.
        __getitem__(index): Return the service object at the specified index,
            allowing indexed access to the collection.

    This design separates the collection of services from the representation
    of an individual service while providing convenient indexed access to
    the mapped service objects.
    """
    @map_fields
    class Service:
        """Represent an individual flight service using descriptor mappings.

        The `map_fields` decorator processes `_nodes` to provide attribute
        access to the service properties defined in the JSON response.

        Attributes:
            code (str): The service code.
            name (str): The service name.
            description (str): A description of the service.
            status (str): The current service status.
        """
        _nodes = [
            ("code", None),
            ("name", None),
            ("description", None),
            ("status", None),
        ]

    def __init__(self, services_info: list):
        """Initialize the collection from a list of service dictionaries.

        Args:
            services_info (list): A list of dictionaries containing the
                JSON data for individual flight services.
        """
        self._services = [self.Service(service) for service in services_info]

    def __getitem__(self, index):
        """Return the service at the specified index.

        Args:
            index: The zero-based index of the service to retrieve.

        Returns:
            Services.Service: The mapped service object at the specified index.

        Raises:
            IndexError: If the index is outside the valid range for the list.
            TypeError: If the index is not a valid list index.
        """
        return self._services[index]
```

### Field descriptor
The `Field` class is a descriptor that provides controlled attribute access to
the underlying JSON data.
It acts as an abstraction layer between the Python object and the dictionary 
containing the JSON response.

```python
class Field:
    """Descriptor for mapping JSON fields to Python object attributes.

    A Field descriptor provides controlled access to a value stored in the
    underlying JSON data. For simple fields, the corresponding JSON value is
    returned directly. For nested JSON objects, ``field_type`` can be
    specified to convert the JSON dictionary into an instance of the
    corresponding Python class.

    Attributes:
        json_node: key/node of the corresponding field in the JSON data.
        _field_type: Optional Python type used to represent a nested JSON
            object.
    """

    def __init__(self, json_node, field_type=None):
        """Initialize a Field descriptor.

        Args:
            json_node: key/node of the field in the underlying JSON data.
            field_type: Optional Python class used to map nested JSON data
                to a Python object.
        """
        self.json_node = json_node
        self._field_type = field_type

    def __get__(self, obj, cls):
        """Retrieve the value associated with the JSON field.

        If accessed through the class, the descriptor itself is returned.
        When accessed through an instance, the corresponding value is
        retrieved from the instance's underlying JSON data. If a
        ``field_type`` is specified, the JSON data is converted into an
        instance of that type.

        Args:
            obj: Instance whose underlying JSON data should be accessed.
            cls: Class through which the descriptor is accessed.

        Returns:
            The corresponding JSON value or an instance of ``field_type``.
        """
        if obj is None:
            return self

        value = obj.__dict__[self.json_node]    

        if self._field_type:
            return self._field_type(value)

        return value

    def __set__(self, obj, value):
        """Prevent modification of the mapped JSON field.

        Args:
            obj: Instance on which the assignment was attempted.
            value: Value that was assigned to the field.

        Raises:
            AttributeError: Always raised because mapped fields are read-only.
        """
        raise AttributeError(f"Field {self.json_node} is read-only")
```
* `json_node` stores the corresponding key/node in the JSON object.
* `field_type` optionally specifies the Python class used to represent a nested JSON object.

### A class decorator that performs dynamic class configuration 
```python
def map_fields(cls):
    """Configure a JSON-backed class using its declared field mappings.
    The decorator reads the class-level ``_nodes`` definition and dynamically
    creates :class:`Field` descriptors for each mapped JSON node. It also
    injects an ``__init__`` method that stores the supplied JSON data on the
    instance.
    Args:
        cls: The class being decorated. The class must define a ``_nodes``
            attribute containing the JSON field mappings.
    Returns:
        type: The configured class with its ``Field`` descriptors and
            generated ``__init__`` method.
    """

    def _mapping(cls, nodes: tuple):
        """Create and attach Field descriptors for the specified nodes.
        Args:
            cls: Class to which the descriptors are added.
            nodes: Iterable containing JSON node names and their corresponding
                Python types.
        """
        for node, mapping in nodes:
            setattr(cls, node, Field(node, field_type=mapping))
    
    _mapping(cls, cls._nodes)   # creating descriptor objects on the class 

    def __init__(self, info: dict):
        """Initialize an instance with its underlying JSON data.
        Args:
            info: Dictionary containing the JSON data represented by the
                object.
        """
        self.__dict__.update(info)  # updating instance dict of obj

    setattr(cls, "__init__", __init__)  # attaching __init__ method on decorated class 
    return cls
```

### Reading the declarative field definition

The `map_fields` function is a class decorator that performs dynamic class configuration . 
It takes a declarative `_nodes` definition and converts each entry into a Field descriptor, 
then injects a common `__init__` implementation into the decorated class.

`map_fields` receives the class object itself as its `cls` argument. The decorator expects the class to 
define `_nodes`,
```python
_nodes = [
    ("passenger", Passenger),
    ("flight", Flight)
]
```
Each tuple contains two pieces of information, `("field_name", field_type)`, for example,
`("first_name", None)`, meaning `first_name` corresponds directly to `first_name` JSON key.
when you say, `("contact", Contact)`, `contact` is the JSON node contains nested data and
should be represented or mapped to `Contact` object. This makes `_nodes` effectively a declarative 
mapping specification.

### Dynamically creating descriptors

The nested `_mapping` function processes the `_nodes` definition.
This function is actually the core of the **metaprogramming** part of the mapper. 
It takes the declarative `_nodes` definition and dynamically adds `Field` descriptors 
to the class.
```python
def _mapping(cls, nodes):
    for node, mapping in nodes:
        setattr(cls, node, Field(node, field_type=mapping))
```

In the above function, the argument `cls` is the actual class or 
the reference of the class that is being configured. 
For example, when `Passenger` class is decorated,
```python
@map_fields
class Passenger:
    _nodes = [
        ("passenger_id", None),
        ("first_name", None),
        ("last_name", None),
        ("contact", Contact),
        ("address", Address),
    ]
```
`cls` refers to the `Passenger` class itself.`nodes` receives the `_nodes` list.

For the tuple `("first_name", None)`, the `_mapping` function maps the `first_name` JSON
node directly to python attribute `first_name`. where as for `("address", Address)` maps
`address` JSON node and represent its nested JSON object using the `Address` class.
The JSON node name and the corresponding Python attribute name are intentionally 
kept identical to maintain a consistent and easily identifiable mapping between the 
JSON structure and the Python object model.

### Importance of setattr

The `setattr` function dynamically binds a `Field` descriptor instance to the target class.
The attribute name is determined at runtime for each `(node, mapping)` tuple defined 
at the class-level `_nodes` collection, enabling the decorator to construct the class's 
field mappings dynamically rather than requiring them to be declared explicitly in the 
class body. For example,
```python
setattr(Passenger, "first_name", Field("first_name", field_type=None))
```
is effectively equivalent to,
```python
Passenger.first_name = Field("first_name")
```
and similarly,
```python
setattr(Passenger, "address", Field("address", field_type=Address))
```
is equivalent to,
```python
Passenger.address = Field("address", field_type=Address)
```
Once the mapping is done, the class looks like,
```python
class Passenger:
    _nodes = [
        ("passenger_id", None),
        ("first_name", None),
        ("last_name", None),
        ("contact", Contact),
        ("address", Address),
    ]

    first_name = Field("first_name")
    last_name = Field("last_name")
    contact = Field("contact", field_type=Contact)
    address = Field("address", field_type=Address)
```
The important point is that **we did not explicitly write those Field assignments**. 
They were generated dynamically at runtime.

### Injecting initializer to class
The decorator also dynamically creates an initializer and attaches it to the class

```python
setattr(cls, "__init__", __init__)
```

is conceptually equivalent to adding
```python
class Passenger:
    def __init__(self, info):
        self.__dict__.update(info)
```
As a result, you don't need to repeat the same initialization logic in every model
```python
@map_fields
class Passenger:
    ...

@map_fields
class Contact:
    ...

@map_fields
class Address:
    ...
```
All the above of them automatically receive the same initialization behavior.

### The complete transformation
```python
@map_fields
class Passenger:
_nodes = [
    ("passenger_id", None),
    ("first_name", None),
    ("last_name", None),
    ("contact", Contact),
    ("address", Address),
]
```
Conceptually, the decorator transforms it into something similar to
```python
class Passenger:
_nodes = [
    ("passenger_id", None),
    ("first_name", None),
    ("last_name", None),
    ("contact", Contact),
    ("address", Address),
]

    first_name = Field("first_name", field_type=None)
    contact = Field("contact", field_type=Contact)
    address = Field("address", field_type=Address)

    def __init__(self, info):
        self.__dict__.update(info)
```

### How this works with the Field descriptor

The decorator itself does not retrieve values from the JSON. It only establishes the mapping.
The actual retrieval is delegated to the `Field` descriptor.

So the responsibilities are clearly separated as shown below,

| Component  | Responsibility                                           |
| ---------- |----------------------------------------------------------|
| `_nodes`   | Declares the JSON-to-Python mapping                      |
| `map_fields` | Dynamically creates the descriptors and injects `__init__` |
| `Field`    | Implements attribute access                              |
| `__get__`  | Retrieves the corresponding JSON value                   |
| `field_type` | Determines whether nested data should be wrapped         |
| `__init__` | Stores the underlying JSON dictionary                    |

The `map_fields` decorator is doing more than simply "decorating" the class in the 
conventional sense. It is modifying the class object at decoration time by dynamically
attaching descriptors and injecting its `__init__` method. 
This is a classic example of Python ***metaprogramming***

***The class declares its data model through `_nodes`, while the decorator generates the 
corresponding class structure dynamically.***

### Why it is interesting design
The real strength of your decorator is that it removes repetitive structural code from the model classes.
Without the decorator, every class would need to explicitly define
```python
field = Field(...)
```
and 
```python
def __init__(self, info):
    self.__dict__.update(info)
```
But With the decorator, the model only describes what the structure is,
```python
_nodes = [
    ("first_name", None),
    ("contact", Contact),
    ("address", Address),
]
```
While the decorator and descriptor infrastructure define how that structure is implemented.

With the descriptor-based field mappings and class decorator in place, 
we can now access the reservation data through a clean, attribute-based interface. 
The following interactive Python session demonstrates how scalar fields, nested objects, 
and collection elements can be accessed without directly navigating the 
underlying JSON dictionaries:
```python
>>> reservations[0].passenger.first_name
'John'
>>> reservations[0].passenger.contact.email
'john.doe@example.com'
>>> reservations[0].services
<__main__.Services object at 0x1057fb7f0>
>>> reservations[0].services[0]
<__main__.Services.Service object at 0x1057fb760>
>>> reservations[0].services[0].name
'Premium Meal'
>>> reservations[0].services[0].status
'CONFIRMED'
>>> reservations[0].services[1].name
'Inflight Wi-Fi'
>>> reservations[0].services[1].status
'CONFIRMED'
```
These lookups demonstrate the abstraction provided by the mapper. 
The client code does not need to know how the underlying JSON dictionaries are
structured or perform explicit key-based lookups. Instead, nested JSON objects are
exposed as corresponding Python objects and can be traversed naturally using 
attribute access.

### Final Thoughts
In this article, we built a lightweight, declarative JSON-to-Object mapper using Python 
descriptors and class decorators. Rather than manually writing repetitive attribute
mappings and nested object construction logic for every model class, we defined the
JSON structure declaratively through the `_nodes` class attribute and allowed the
mapping infrastructure to construct the required descriptors dynamically.

The Field descriptor is responsible for controlling attribute access and resolving values
from the underlying JSON data, while the `map_fields` class decorator dynamically 
configures each model class based on its declared field mappings. 
This separation of responsibilities keeps the model classes concise while centralizing 
the mapping behavior in reusable components.

One of the most interesting aspects of this approach is the use of Python metaprogramming.
The classes describe what needs to be mapped, while the decorator and descriptors determine
how that mapping is implemented. The use of `setattr` to dynamically attach descriptors 
demonstrates how Python classes can be inspected and modified programmatically at runtime.

The resulting object model provides a clean and intuitive interface for navigating
deeply nested JSON structures. Instead of exposing dictionaries and requiring callers
to perform repeated key-based lookups.


[Previous Article](../json/reading_json.md) \| [Next Article](../logging/logging.md)