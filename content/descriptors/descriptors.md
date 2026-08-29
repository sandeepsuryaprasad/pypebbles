## Building a Declarative JSON-to-Object Mapper — A Metaprogramming Approach

**Last Updated:** August 15, 2026

When working with real-world APIs, JSON responses are often far more complex than 
simple key-value structures. A single response can contain deeply nested objects, 
collections, optional fields, and multiple levels of related data. Accessing such 
data directly through dictionary lookups can quickly become verbose, difficult to read,
and tightly coupled to the structure of the response. 

In one of the projects I worked on in the airline domain, we had to work with complex
JSON responses containing information about reservations, passengers, flights, 
airports, aircraft, baggage, and several other related entities. As the response 
structure became more complex, accessing and working with the data through conventional 
dictionary lookups became increasingly difficult to read and maintain.

This article presents a real-world scenario inspired by that project and demonstrates how I approached the problem using Python's object-oriented capabilities.
The solution uses Python descriptors to map fields from the JSON response to Python attributes and to transparently construct objects for nested structures.

Let us consider the following JSON response. For this demonstration, 
we will work with a representative JSON containing reservation and 
associated passenger information. The source JSON is structured as a list of 
reservation records, with each record representing the reservation details of an 
individual passenger. To keep the example concise and focused, we will use a 
single reservation record from that collection.


<details>
<summary><strong> Click here to expand/collapse complete JSON response</strong></summary>

<div markdown="1">

`reservations.json`
```json
[
    {
      "reservation": {
        "confirmation_number": "X7K9PQ",
        "booking_status": "CONFIRMED",
        "booking_date": "2026-08-10T14:32:18-05:00",
        "ticket_status": "TICKETED",
        "currency": "USD"
      },
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
        },
        "frequent_flyer": {
          "program": "Spam Airlines dvantage",
          "membership_number": "SA987654321",
          "status": "Executive Platinum",
          "miles_balance": 184250
        }
      },
      "flight": {
        "airline": {
          "code": "SA",
          "name": "Spam Airlines",
          "headquarters": {
            "city": "Austin",
            "state": "Texas",
            "state_code": "TX"
          }
        },
        "flight_number": "SA1234",
        "flight_status": "SCHEDULED",
        "aircraft": {
          "registration": "N003AN",
          "manufacturer": "Boeing",
          "model": "737-800",
          "configuration": "160"
        },
        "departure": {
          "airport": {
            "code": "AUS",
            "name": "Austin-Bergstrom International Airport",
            "city": "Austin",
            "state": "Texas",
            "state_code": "TX",
            "country": "United States",
            "terminal": "South Terminal"
          },
          "scheduled": {
            "date": "2026-09-15",
            "time": "08:30",
            "timezone": "America/Chicago"
          },
          "gate": "34",
          "boarding_time": "07:50"
        },
        "arrival": {
          "airport": {
            "code": "JFK",
            "name": "John F. Kennedy International Airport",
            "city": "New York",
            "state": "New York",
            "state_code": "NY",
            "country": "United States",
            "terminal": "Terminal 8"
          },
          "scheduled": {
            "date": "2026-09-15",
            "time": "13:25",
            "timezone": "America/New_York"
          },
          "gate": "B22"
        },
        "duration": {
          "hours": 3,
          "minutes": 55
        },
        "distance": {
          "value": 1511,
          "unit": "miles"
        }
      },
      "seat": {
        "number": "12A",
        "class": "Business",
        "cabin": "Business",
        "position": "Window",
        "is_exit_row": false,
        "is_extra_legroom": true
      },
      "baggage": {
        "checked": {
          "allowed_pieces": 2,
          "weight_limit": {
            "value": 50,
            "unit": "lbs"
          }
        },
        "carry_on": {
          "allowed_pieces": 1,
          "weight_limit": {
            "value": 40,
            "unit": "lbs"
          }
        },
        "personal_item": {
          "allowed": true,
          "description": "One small personal item"
        }
      },
      "payment": {
        "status": "PAID",
        "method": {
          "type": "CREDIT_CARD",
          "provider": "Visa"
        },
        "fare": {
          "base_fare": 485,
          "taxes": 72.75,
          "airport_fees": 18.4,
          "service_fee": 25,
          "total": 601.15
        }
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
      ],
      "emergency_contact": {
        "name": "Sarah Emergency",
        "relationship": "Spouse",
        "phone": "+1-000-999-0199",
        "address": {
          "street": "890 Another Example Avenue",
          "suite": "Suite 205",
          "city": "Springfield",
          "state": "Illinois",
          "state_code": "IL",
          "zip_code": "62700",
          "country": "United States"
        }
      },
      "notifications": {
        "email": {
          "enabled": true,
          "address": "john.doe@example.com"
        },
        "sms": {
          "enabled": true,
          "phone": "+1-000-000-0000"
        },
        "push": {
          "enabled": false
        }
      }
    }
]
```

</div>
</details>

**Disclaimer:** The JSON data used in this article is for demonstration purposes. It does not represent actual production data, 
and any resemblance to real-world data is purely coincidental.

Below is the hierarchical structure of the `reservations.json` response. 
Let us break down the response structure to understand its composition.
The root response object contains nine top-level nodes: `reservation`, `passenger`, 
`flight`, `seat`, `baggage`, `payment`, `services`, `emergency_contact`, and `notifications`.
Several of these top-level nodes contain nested objects, forming a hierarchical 
JSON structure. For example, the `passenger` node contains child objects such as 
`contact`, `address`, and `frequent_flyer`. These nested objects, in turn, contain 
their own attributes and, in some cases, additional nested objects.

<details>
<summary><strong> Click here to expand/collapse complete JSON tree</strong></summary>

<div markdown="1">

```commandline
ReservationInfo
│
├── reservation
│   ├── confirmation_number
│   ├── booking_status
│   ├── booking_date
│   ├── ticket_status
│   └── currency
│
├── passenger
│   ├── passenger_id
│   ├── first_name
│   ├── last_name
│   ├── contact
│   │   ├── email
│   │   ├── phone
│   │   └── alternate_phone
│   ├── address
│   │   ├── city
│   │   ├── state
│   │   └── country
│   └── frequent_flyer
│       ├── program
│       ├── membership_number
│       ├── status
│       └── miles_balance
│
├── flight
│   ├── airline
│   │   ├── code
│   │   ├── name
│   │   └── headquarters
│   │       ├── city
│   │       ├── state
│   │       └── state_code
│   ├── flight_number
│   ├── flight_status
│   ├── aircraft
│   │   ├── registration
│   │   ├── manufacturer
│   │   ├── model
│   │   └── configuration
│   ├── departure
│   │   ├── airport
│   │   │   ├── code
│   │   │   ├── name
│   │   │   ├── city
│   │   │   ├── state
│   │   │   ├── state_code
│   │   │   ├── country
│   │   │   └── terminal
│   │   ├── scheduled
│   │   │   ├── date
│   │   │   ├── time
│   │   │   └── timezone
│   │   ├── gate
│   │   └── boarding_time
│   ├── arrival
│   │   ├── airport
│   │   │   ├── code
│   │   │   ├── name
│   │   │   ├── city
│   │   │   ├── state
│   │   │   ├── state_code
│   │   │   ├── country
│   │   │   └── terminal
│   │   ├── scheduled
│   │   │   ├── date
│   │   │   ├── time
│   │   │   └── timezone
│   │   └── gate
│   ├── duration
│   │   ├── hours
│   │   └── minutes
│   └── distance
│       ├── value
│       └── unit
│
├── seat
│   ├── number
│   ├── class
│   ├── cabin
│   ├── position
│   ├── is_exit_row
│   └── is_extra_legroom
│
├── baggage
│   ├── checked
│   │   ├── allowed_pieces
│   │   └── weight_limit
│   │       ├── value
│   │       └── unit
│   ├── carry_on
│   │   ├── allowed_pieces
│   │   └── weight_limit
│   │       ├── value
│   │       └── unit
│   └── personal_item
│       ├── allowed
│       └── description
│
├── payment
│   ├── status
│   ├── method
│   │   ├── type
│   │   └── provider
│   └── fare
│       ├── base_fare
│       ├── taxes
│       ├── airport_fees
│       ├── service_fee
│       └── total
│
├── services [array]
│   └── Service
│       ├── code
│       ├── name
│       ├── description
│       └── status
│
├── emergency_contact
│   ├── name
│   ├── relationship
│   ├── phone
│   └── address
│       ├── street
│       ├── suite
│       ├── city
│       ├── state
│       ├── state_code
│       ├── zip_code
│       └── country
│
└── notifications
    ├── email
    │   ├── enabled
    │   └── address
    ├── sms
    │   ├── enabled
    │   └── phone
    └── push
        └── enabled
```
</div>
</details>


Let us implement the `Reservations` class, which serves as the **data access layer** 
responsible for loading and deserializing reservation data from the JSON file
and making it available through a structured Python interface.

```python
from json import load
from pathlib import Path


class Reservations:
    """Provide sequence-style access to reservation information.

    Loads reservation data from ``reservation.json`` and converts each
    reservation record into a :class:`Reservation` object. The resulting
    collection supports sequence-style operations such as indexed access
    and retrieving the number of reservations.

    Attributes:
        _path: Path to the JSON file containing the reservation data.
        _data: Parsed reservation data loaded from the JSON file.
        _reservations: List of :class:`Reservation` objects created from
            the parsed reservation data.
    """

    def __init__(self):
        """Initialize the Reservations collection.

        Resolves the reservation JSON file path, loads and deserializes
        the reservation data, and creates the corresponding
        :class:`Reservation` objects.
        """
        self._path = self._json_file_path
        self._data = self._load_json_data
        self._reservations = self._get_reservations

    @property
    def _json_file_path(self) -> Path:
        """Return the path to the reservation JSON file.

        Returns:
            Path: Path to ``reservation.json``.

        Raises:
            FileNotFoundError: If ``reservation.json`` does not exist.
        """
        path = Path("reservations.json")
        if not path.exists():
            raise FileNotFoundError(f"{path} does not exist")
        return path

    @property
    def _load_json_data(self):
        """Load and deserialize reservation data from the JSON file.

        Reads the JSON file and deserializes its contents into the
        corresponding Python representation using :func:`json.load`.

        Returns:
            The parsed reservation data, typically a list of dictionaries
            representing reservation records.
        """
        with open(self._path, "r") as json_file:
            return load(json_file)

    @property
    def _get_reservations(self):
        """Convert reservation records into Reservation objects.

        Iterates over the parsed reservation data and creates a
        :class:`Reservation` object for each reservation record.

        Returns:
            list[Reservation]: A list of :class:`Reservation` objects
                created from the parsed reservation data.
        """
        return [Reservation(reservation) for reservation in self._data]

    def __getitem__(self, index):
        """Return the reservation at the specified index.

        Provides sequence-style indexed access to the collection of
        :class:`Reservation` objects.

        Args:
            index: Zero-based index of the reservation to retrieve.

        Returns:
            Reservation: The reservation object at the specified index.

        Raises:
            IndexError: If the specified index is outside the valid range.
        """
        return self._reservations[index]

    def __len__(self):
        """Return the number of reservations in the collection.

        Enables the use of the built-in :func:`len` function on a
        ``Reservations`` instance.

        Returns:
            int: Number of :class:`Reservation` objects in the collection.
        """
        return len(self._reservations)
```
We can now instantiate the `Reservations` class to load the reservation data and access 
individual reservation records through the resulting collection.
```python
>>> reservations = Reservations()
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
we will implement only the `Passenger`, and `Services` classes. 
Each JSON node is mapped to its corresponding Python attribute through the `Field` 
descriptor. This provides a sufficient foundation to demonstrate how descriptors can 
abstract the underlying JSON structure and expose both scalar values and nested 
objects through a structured, attribute-based interface.

```python
@map_fields
class Passenger:
    """Represent passenger information using descriptor-based field mappings.

    The class uses the :func:`map_fields` decorator to dynamically create
    :class:`Field` descriptors for the JSON nodes declared in ``_nodes``.
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
        JSON field, while ``passenger.address.city`` provides attribute-based
        access to the nested address data.
    """

    _nodes = [
        ("passenger_id", None),
        ("title", None),
        ("first_name", None),
        ("middle_name", None),
        ("last_name", None),
        ("date_of_birth", None),
        ("gender", None),
        ("contact", Contact),
        ("address", Address),
        ("frequent_flyer", FrequentFlyer),
    ]
```

```python
@map_fields
class Contact:
    """Represent passenger contact information using descriptor mappings.

    The class uses the :func:`map_fields` decorator to dynamically create
    :class:`Field` descriptors for the contact-related JSON nodes declared
    in ``_nodes``. Each field represents a scalar value in the underlying
    JSON data and is therefore mapped directly without additional object
    conversion.

    Attributes:
        _nodes: Mapping of contact JSON field names to their corresponding
            Python representations.
    """

    _nodes = [
        ("email", None),
        ("phone", None),
        ("alternate_phone", None),
    ]
```

```python
class Services:
    """Represent a collection of services selected for a reservation.

    Encapsulates the ``services`` JSON array and converts each service
    dictionary into a structured :class:`Service` object. The nested
    :class:`Service` class represents the individual service entries,
    while ``Services`` provides indexed access to the collection.

    Attributes:
        _data: List of :class:`Service` objects created from the JSON service
            records.
    """

    @map_fields
    class Service:
        """Represent an individual service associated with a reservation.

        Uses the :func:`map_fields` decorator to create :class:`Field`
        descriptors for the scalar attributes defined in ``_nodes``.

        Attributes:
            _nodes: Mapping of service JSON fields to their corresponding
                Python representation.
        """

        _nodes = [
            ("code", None),
            ("name", None),
            ("description", None),
            ("status", None),
        ]

    def __init__(self, services_info):
        """Initialize the service collection from JSON data.

        Converts each service dictionary into a :class:`Service` object.

        Args:
            services_info: List of dictionaries containing service
                information.
        """
        self._services = [self.Service(service) for service in services_info]

    def __getitem__(self, index):
        """Return the service at the specified index.

        Provides sequence-style indexed access to the individual
        :class:`Service` objects.

        Args:
            index: Zero-based index of the service to retrieve.

        Returns:
            Service: The service object at the specified index.

        Raises:
            IndexError: If the specified index is outside the valid range.
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
        name: Name of the corresponding field in the JSON data.
        _field_type: Optional Python type used to represent a nested JSON
            object.
    """

    def __init__(self, json_node, field_type=None):
        """Initialize a Field descriptor.

        Args:
            name: Name of the field in the underlying JSON data.
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

### A class Decorator that performs dynamic class configuration 
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

    def _mapping(cls, nodes):
        """Create and attach Field descriptors for the specified nodes.
        Args:
            cls: Class to which the descriptors are added.
            nodes: Iterable containing JSON node names and their corresponding
                Python types.
        """
        for node, mapping in nodes:
            setattr(cls, node, Field(node, field_type=mapping))
    
    # calling _mapping function that creates descriptor objects on the class
    _mapping(cls, cls._nodes) 

    def __init__(self, info):
        """Initialize an instance with its underlying JSON data.
        Args:
            info: Dictionary containing the JSON data represented by the
                object.
        """
        self.__dict__.update(info)  # updating instance dict of obj

    setattr(cls, "__init__", __init__)  # defining __init__ method on decorated class 
    return cls
```

### Reading the declarative field definition

The `map_fields` function is a class decorator that performs dynamic class configuration . 
It takes a declarative _nodes definition and converts each entry into a Field descriptor, 
then injects a common `__init__` implementation into the decorated class.

`map_fields` receives the class object itself as its `cls` argument. The decorator expects the class to 
define _nodes,
```python
_nodes = [
    ("passenger_id", None),
    ("title", None),
    ("first_name", None),
    ("middle_name", None),
    ("last_name", None),
    ("date_of_birth", None),
    ("gender", None),
    ("contact", Contact),
    ("address", Address),
    ("frequent_flyer", FrequentFlyer),
]
```
Each tuple contains two pieces of information, `("field_name", field_type)`, for example,
`("first_name", None)`, meaning `first_name` corresponds directly to `first_name` JSON key.
when you say, `("contact", Contact)`, `contact` is the JSON node contains nested data and
should be represented or mapped to `Contact` object. This makes _nodes effectively a declarative 
mapping specification.

### Dynamically creating descriptors

The nested `_mapping` function processes the `_nodes` definition:
```python
def _mapping(cls, nodes):
    for node, mapping in nodes:
        setattr(cls, node, Field(node, field_type=mapping))
```
For example `("first_name", None)` the following operation effectively occurs,
```python
setattr(Passenger, "first_name", Field("first_name", field_type=None))
```
which is equivalent to dynamically adding `Passenger.first_name = Field("first_name", None)`
Similarly, when you say `("contact", Contact)` the following operation occurs,
```python
Passenger.contact = Field("contact", field_type=Contact)
```
So basically you only declare the mappings in `_nodes`

### Importance of _setattr_

The `setattr` dynamically adds an attribute to the class. When you say 
`setattr(cls, node, Field(node, field_type=mapping))`, 
Conceptually, `setattr(Passenger, "first_name", descriptor)` is equivalent to 
`Passenger.first_name = descriptor`. The important difference is that the attribute name is 
determined at runtime from `_nodes`

### Injecting _ _ init_ _ to class

The decorator also dynamically creates an initializer and attaches it to the class
```python
def __init__(self, info):
    self.__dict__.update(info)
```

`setattr(cls, "__init__", __init__)` is conceptually equivalent to adding
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
        ("first_name", None),
        ("contact", Contact),
    ]
```
Conceptually, the decorator transforms it into something similar to
```python
class Passenger:
    _nodes = [
        ("first_name", None),
        ("contact", Contact),
    ]

    first_name = Field("first_name", field_type=None)
    contact = Field("contact", field_type=Contact)

    def __init__(self, info):
        self.__dict__.update(info)
```

### How this works with the _Field_ descriptor

The decorator itself does not retrieve values from the JSON. It only establishes the mapping
The actual retrieval is delegated to the Field descriptor.

```python
def __get__(self, obj, cls):
    if obj is None:
        return self

    value = obj._data[self.name]

    if self._field_type:
        return self._field_type(value)

    return value
```
So the responsibilities are clearly separated as shown below,

| Component  | Responsibility                                   |
| ---------- | ------------------------------------------------ |
| `_nodes`   | Declares the JSON-to-Python mapping              |
| `map_fields` | Dynamically creates the descriptors            |
| `Field`    | Implements attribute access                      |
| `__get__`  | Retrieves the corresponding JSON value           |
| `field_type` | Determines whether nested data should be wrapped |
| `__init__` | Stores the underlying JSON dictionary            |

The `map_fields` decorator is doing more than simply "decorating" the class in the 
conventional sense. It is modifying the class object at decoration time by dynamically
attaching descriptors and replacing/injecting its `__init__` method. 
In other words, this is a classic example of Python metaprogramming

***The class declares its data model through `_nodes`, while the decorator generates the 
corresponding class structure dynamically.***



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
Here, nested attributes are accessed using dot notation, while individual service 
records are accessed using standard sequence indexing since there can be one or more services.


Back to  [Articles](../articles.md)