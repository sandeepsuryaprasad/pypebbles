## Metaprogramming with Python Descriptors and Class Decorators - Building a declarative JSON-to-Object Mapper

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

For the purpose of this demonstration and to keep the article concise, we will implement 
only the `Passenger`, `Contact`, and `Services` classes. 
This is sufficient to demonstrate how the descriptor-based model provides structured, 
attribute-based access to both scalar and nested JSON data, as illustrated below,

### Field descriptor
The `Field` class is a descriptor that provides controlled attribute access to
the underlying JSON data.
It acts as an abstraction layer between the Python object and the dictionary 
containing the JSON response.

```python
class Field:
    """Descriptor that maps a Python attribute to a field in JSON data.

    Provides read-only attribute access to values stored in the underlying
    JSON dictionary. When a ``field_type`` is specified, the retrieved JSON
    value is wrapped in the corresponding Python class, allowing nested JSON
    objects to be represented as structured Python objects.
    """

    def __init__(self, json_node, field_type=None):
        """Initialize a Field descriptor.

        Args:
            name: Name of the corresponding key in the JSON data.
            field_type: Optional Python type used to wrap nested JSON data.
                If omitted, the JSON value is returned directly.
        """
        self.json_node = json_node
        self._field_type = field_type

    def __get__(self, obj, cls):
        """Retrieve the value associated with the mapped JSON field.

        When accessed through an instance, the method performs a dictionary
        lookup using the configured field name. If a ``field_type`` is
        specified, the retrieved value is converted into an instance of that
        type. When accessed through the class, the descriptor itself is
        returned.

        Args:
            obj: Instance through which the descriptor is accessed.
            cls: Class that owns the descriptor.

        Returns:
            The corresponding JSON value, or an instance of ``field_type``
            when a mapped type is specified.
        """
        if obj is None:
            return self

        json_data = obj._data[self.json_node]

        if self._field_type:
            return self._field_type(json_data)

        return json_data

    def __set__(self, obj, value):
        """Prevent modification of the mapped JSON field.

        Args:
            obj: Instance whose field is being modified.
            value: Value that was supplied for the assignment.

        Raises:
            AttributeError: Always raised because mapped fields are read-only.
        """
        raise AttributeError(f"Field {self.json_node} is read-only")
```
* `json_node` stores the corresponding key/node in the JSON object.
* `field_type` optionally specifies the Python class used to represent a nested JSON object.



Back to  [Articles](../articles.md)