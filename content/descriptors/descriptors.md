## Python Descriptors - Building a Reusable JSON Field Mapping Framework
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

Let's consider below JSON response, For this demonstration, we will work with a 
representative JSON response containing the reservation and associated information 
for a single passenger. 

<details>
<summary><strong>▸ Click here to expand/collapse complete JSON response</strong></summary>

<div markdown="1">

`reservations.json`
```json
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
<summary><strong>▸ Click here to expand/collapse complete JSON tree</strong></summary>

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


Let us re-use the code that I wrote in previous article _From JSON Data to Python Objects_ with 
some minor modifications.
```python
from json import load
from pathlib import Path

class FlightReservation:
    def __init__(self):
        self._path = self._json_file_path
        self._data = self._load_json_data
        self._passenger = None

    @property
    def _json_file_path(self):
        path = Path("reservation.json")
        if not path.exists():
            raise FileNotFoundError(f"{path} does not exist")
        return path

    @property
    def _load_json_data(self):
        with open(self._path, "r") as json_file:
            json_object = load(json_file)
            return json_object

    @property
    def info(self):
        if not self._passenger:
            self._passenger = ReservationInfo(self._data)
        return self._passenger
```
The `info` property serves as the public interface to the underlying `ReservationInfo`
object, providing controlled and lazy access to the reservation data.

Now, let us introduce a layer of abstraction for nine top-level 
nodes, encapsulating each section of the JSON response behind a
dedicated Python object.  

```python
class ReservationInfo:
    reservation = Field("reservation", field_type=Reservation)
    passenger = Field("passenger", field_type=Passenger)
    flight = Field("flight", field_type=Flight)
    seat = Field("seat", field_type=Seat)
    baggage = Field("baggage", field_type=Baggage)
    payment = Field("payment", field_type=Payment)
    services = Field("services")
    emergency_contact = Field("emergency_contact", field_type=EmergencyContact)
    notifications = Field("notifications", field_type=Notifications)

    def __init__(self, reservation_info):
        self._data = reservation_info
```
In the `ReservationInfo` class, attributes such as `reservation`, `passenger`,
`flight`, `seat`, and `payment` are class attributes whose values are instances 
of the `Field` descriptor.
### Field descriptor
The `Field` class is a descriptor that provides controlled attribute access to
the underlying JSON data.
It acts as an abstraction layer between the Python object and the dictionary 
containing the JSON response.
```python
class Field:
    def __init__(self, name, field_type=None):
        self.name = name
        self._field_type = field_type

    def __get__(self, obj, cls):
        if obj is None:
            return self
        json_data = obj._data[self.name]
        if self._field_type:
            return self._field_type(json_data)
        return json_data

    def __set__(self, obj, value):
        raise AttributeError(f"Field {self.name} is read-only")
```
* `name` stores the corresponding key in the JSON object.
* `field_type` optionally specifies the Python class used to represent a 
nested JSON object.

when we say `flight = Field("flight", Flight)` the `flight` attribute maps to
the `"flight"` key in the JSON data and should be represented by a Flight object because
when `"flight"` key is accessed, the resulting value is one more nested JSON object.

`__get__()` is invoked automatically whenever the corresponding attribute 
is accessed. For example,
```commandline
>>> reservation = FlightReservation()
>>> reservation.info.flight     # Causes python to invoke __get__ method in Field descriptor
```
Here is what happens when `__get__`, is invoked, 
1. Retrieves the corresponding value from `obj._data`
2. If `field_type` is specified, wraps that dictionary in the specified class.
3. Otherwise, returns the value directly. (meaning there is no nested JSON object)

`__set__()` makes the descriptor read-only. So you cannot be doing something like,
```commandline
>>> reservation.info.flight = 1234
```
The `Field` descriptor provides three important capabilities:
* **Attribute mapping** - maps Python attributes to JSON keys.
* **Nested object conversion** - converts nested dictionaries into appropriate Python objects.
* **Read-only access** - prevents modification of the underlying JSON data.

This allows code to navigate a deeply nested JSON response using normal 
Python attribute access, while the descriptor handles the underlying dictionary
access and object creation transparently.

### Implementing the Mapped Types
For the purpose of demonstration, I will be implementing three Mapped types,
`Passenger`, `Address` and `Services`. For nested JSON objects, 
the Field descriptor will be configured with the corresponding mapped type so
that the nested dictionary is automatically converted into an
instance of that type.

Let us implement a mapped type `Passenger` class,
```python
class Passenger: 
    first_name = Field("first_name")
    last_name = Field("last_name")
    address = Field("address", field_type=Address) 

    def __init__(self, passenger_info):
        self._data = passenger_info
```
So now when you say `reservation.info.passenger` it returns instance of `Passenger` class.
```commandline
>>> reservation.info.passenger
<__main__.Passenger object at 0x104542790>
```
Now you can access all attributes of `Passenger` class
```commandline
>>> reservation.info.passenger.first_name
'John'
>>> reservation.info.passenger.last_name
'Doe'
```
But accessing `address` will not return the actual value of corresponding node 
in JSON, but rather returns an instance of `Address` class. 
Because `address` node is a nested JSON object which we have abstracted in a 
different class `Address`.

If the JSON node has a nested attribute, we are going pass the class reference to
`Field` descriptor through argument `field_type`, which then the descriptor returns
instance of the class that is passed.
```commandline
>>> reservation.info.passenger.address
<__main__.Address object at 0x104542820>
```
Let us implement `Address` mapped class.
```python
class Address:
    city = Field("city")
    state = Field("state")
    country = Field("country")

    def __init__(self, address_info):
        self._data = address_info
```
You can access the nested `address` JSON nodes through their 
corresponding Python attributes as shown below.
```commandline
>>> reservation.info.passenger.address.city
'Sampleville'
>>> reservation.info.passenger.address.state
'California'
```
When you access `services` on the `ReservationInfo` object, it returns a list 
containing the services selected by the passenger. Each element in the list is
represented as a dictionary object containing the details of an individual service.
```commandline
>>> reservation.info.services
[{'code': 'MEAL', 'name': 'Premium Meal', 'description': 'Chicken and roasted vegetables', 'status': 'CONFIRMED'}, {'code': 'WIFI', 'name': 'Inflight Wi-Fi', 'description': 'High-speed internet access', 'status': 'CONFIRMED'}, {'code': 'LOUNGE', 'name': 'Elite Club', 'description': 'Airport lounge access', 'status': 'CONFIRMED'}]
>>> 
```
Now If you want to access the list of services that the passenger has opted for,
you should index the list.
```commandline
>>> reservation.info.services[0]
{'code': 'MEAL', 'name': 'Premium Meal', 'description': 'Chicken and roasted vegetables', 'status': 'CONFIRMED'}
>>> 
>>> reservation.info.services[1]
{'code': 'WIFI', 'name': 'Inflight Wi-Fi', 'description': 'High-speed internet access', 'status': 'CONFIRMED'}
>>> 
>>> reservation.info.services[2]
{'code': 'LOUNGE', 'name': 'Elite Club', 'description': 'Airport lounge access', 'status': 'CONFIRMED'}
>>> 
```
Suppose if we wanted to access the service at 0th index, we would do something like this,
```commandline
>>> reservation.info.services[0]["code"]
'MEAL'
>>> reservation.info.services[0]["name"]
'Premium Meal'
```
Again, we encounter the same limitation. When we access `reservation.info.services[0]`, 
the expression returns the element at index 0, which is still a dictionary object.
Therefore, we must use either dictionary indexing or the `get()` method to access 
the individual values within that object. A more intuitive and object-oriented approach 
would be to expose these values as attributes,
```commandline
>>> reservation.info.services[0].code
'MEAL'
>>> reservation.info.services[0].name
'Premium Meal'
```
To achieve this, let us introduce one more layer of abstraction, `Services`
```python
class Services:
    class Service:
        code = Field("code")
        name = Field("name")
        description = Field("description")
        status = Field("status")

        def __init__(self, service_info):
            self._data = service_info

    def __init__(self, services_info):
        """Convert list of dicts to Service object"""
        self._data = [self.Service(service) for service in services_info]

    def __getitem__(self, index):
        if index > len(self._data) - 1:
            raise IndexError(f"Service index must be less than {len(self._data)}")
        return self._data[index]
```
The `Services` class is responsible for transforming the services JSON array into a
collection of Python objects. Since each element in the JSON array represents an 
individual service, the implementation uses a nested Service class to model the 
structure of each `service` item. (It is not necessary to have `Service` class
nested inside `Services`. You can choose to have `Service` class outside `Services` class)

The nested `Service` class represents one service entry from the JSON array. 
Its attributes are defined using the `Field` descriptor

The descriptor maps each Python attribute to its corresponding key in the service dictionary.

For example: `"code": "MEAL"` is exposed as `service.code`.

### Converting dictionaries into `Service` objects
```python
def __init__(self, services_info):
    self._data = [self.Service(service) for service in services_info]
```
The list comprehension iterates over each dictionary in `services_info` and 
creates a `Service` object for it.

The `__getitem__` method makes Services behave like a sequence. As a result, 
callers can access an individual service using familiar indexing syntax

Let us map `services` attribute in `ReservationInfo` class to `Service`
```python
class ReservationInfo:
    ...
    # Other attributes omitted
    services = Field("services", field_type=Services)
    ...
```
Now the individual service attributes can then be accessed using normal 
attribute notation.
```python
>>> reservation.info.services[0].code
'MEAL'
>>> reservation.info.services[0].name
'Premium Meal'
>>> reservation.info.services[0].status
'CONFIRMED'
>>> reservation.info.services[1].name
'Inflight Wi-Fi'
>>> reservation.info.services[1].code
'WIFI'
>>> reservation.info.services[1].status
'CONFIRMED'
```
The final implementation can be structured as shown below.
```python
from json import load
from pathlib import Path


class FlightReservation:
    def __init__(self):
        """Initialize the flight reservation and load reservation data."""
        self._path = self._json_file_path
        self._data = self._load_json_data
        self._info = None

    @property
    def _json_file_path(self):
        """Return the path to the reservation JSON file.

        Raises:
            FileNotFoundError: If the reservation JSON file does not exist.

        Returns:
            Path: Path object representing the reservation JSON file.
        """
        path = Path("reservation.json")
        if not path.exists():
            raise FileNotFoundError(f"{path} does not exist")
        return path

    @property
    def _load_json_data(self):
        """Load and return the reservation data from the JSON file.

        Returns:
            dict: Parsed reservation data from the JSON file.
        """
        with open(self._path, "r") as json_file:
            json_object = load(json_file)
            return json_object

    @property
    def info(self):
        """Return the reservation information as a ReservationInfo object.

        The ReservationInfo object is created lazily and cached for
        subsequent accesses.

        Returns:
            ReservationInfo: Object representing the reservation data.
        """
        if not self._info:
            self._info = ReservationInfo(self._data)
        return self._info
```
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

    def __init__(self, name, field_type=None):
        """Initialize a Field descriptor.

        Args:
            name: Name of the field in the underlying JSON data.
            field_type: Optional Python class used to map nested JSON data
                to a Python object.
        """
        self.name = name
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

        json_data = obj._data[self.name]

        if self._field_type:
            return self._field_type(json_data)

        return json_data

    def __set__(self, obj, value):
        """Prevent modification of the mapped JSON field.

        Args:
            obj: Instance on which the assignment was attempted.
            value: Value that was assigned to the field.

        Raises:
            AttributeError: Always raised because mapped fields are read-only.
        """
        raise AttributeError(
            f"Field {self.name} is read-only"
        )
```
```python
class Services:
    """Represent the collection of services associated with a reservation.

    Encapsulates a collection of :class:`Service` objects created from the
    corresponding list of service dictionaries in the JSON response.

    The nested ``Service`` class represents an individual service entry.
    ``Services`` also implements indexed access through ``__getitem__()``,
    allowing individual services to be accessed using standard sequence
    notation.

    Attributes:
        _data: List containing the mapped :class:`Service` objects.
    """

    class Service:
        """Represent an individual service selected by the passenger.

        Attributes:
            code: Unique code identifying the service.
            name: Name of the service.
            description: Description of the service.
            status: Current status of the service.
        """

        code = Field("code")
        name = Field("name")
        description = Field("description")
        status = Field("status")

        def __init__(self, service_info):
            """Initialize a Service object from service data.

            Args:
                service_info: Dictionary containing the service details.
            """
            self._data = service_info

    def __init__(self, services_info):
        """Initialize a Services collection from service data.

        Each dictionary in ``services_info`` is converted into a
        :class:`Service` object and stored internally.

        Args:
            services_info: List of dictionaries containing service details.
        """
        self._data = [self.Service(service) for service in services_info]

    def __getitem__(self, index):
        """Return the service at the specified index.

        Args:
            index: Zero-based index of the service to retrieve.

        Returns:
            Service: The service object at the specified index.

        Raises:
            IndexError: If the specified index is outside the valid range.
        """
        if index > len(self._data) - 1:
            raise IndexError(
                f"Service index must be less than {len(self._data)}"
            )

        return self._data[index]
```
```python
class Address:
    """Represent a passenger's address information.

    Encapsulates the address details associated with a passenger, including
    street, city, state, postal code, and country information.

    Attributes:
        street: Street address.
        suite: Apartment, suite, or unit information.
        city: City associated with the address.
        state: State or administrative region.
        state_code: Abbreviated state or region code.
        zip_code: Postal or ZIP code.
        country: Country associated with the address.
    """

    street = Field("street")
    suite = Field("suite")
    city = Field("city")
    state = Field("state")
    state_code = Field("state_code")
    zip_code = Field("zip_code")
    country = Field("country")

    def __init__(self, address_info):
        """Initialize an Address object from address data.

        Args:
            address_info: Dictionary containing the address information.
        """
        self._data = address_info
```
```python
class Passenger:
    """Represent passenger information from a reservation response.

    Encapsulates the passenger's personal information and provides access
    to related contact, address, and frequent-flyer information through
    mapped Python objects.

    Attributes:
        passenger_id: Unique identifier assigned to the passenger.
        title: Passenger's title, such as Mr, Mrs, or Ms.
        first_name: Passenger's first name.
        middle_name: Passenger's middle name.
        last_name: Passenger's last name.
        date_of_birth: Passenger's date of birth.
        gender: Passenger's gender.
        contact: Passenger's contact information represented by a
            :class:`Contact` object.
        address: Passenger's address represented by an :class:`Address`
            object.
        frequent_flyer: Passenger's frequent-flyer information represented
            by a :class:`FrequentFlyer` object.
    """

    passenger_id = Field("passenger_id")
    first_name = Field("first_name")
    last_name = Field("last_name")
    contact = Field("contact", field_type=Contact)
    address = Field("address", field_type=Address)
    frequent_flyer = Field("frequent_flyer", field_type=FrequentFlyer)

    def __init__(self, passenger_info):
        """Initialize a Passenger object from passenger data.

        Args:
            passenger_info: Dictionary containing passenger details,
                including contact, address, and frequent-flyer information.
        """
        self._data = passenger_info
```
```python
class ReservationInfo:
    """Represent the complete flight reservation information.

    Provides a structured Python interface to the top-level nodes of the
    reservation JSON response. Each field maps a JSON node to its
    corresponding Python object, allowing nested reservation data to be
    accessed through attribute notation.

    Attributes:
        reservation: Reservation and booking information.
        passenger: Passenger details and contact information.
        flight: Flight, airline, aircraft, departure, and arrival details.
        seat: Seat assignment and seating information.
        baggage: Baggage allowance and related information.
        payment: Payment status, payment method, and fare details.
        services: Collection of services selected by the passenger.
        emergency_contact: Emergency contact information.
        notifications: Email, SMS, and push notification preferences.
    """

    reservation = Field("reservation", field_type=Reservation)
    passenger = Field("passenger", field_type=Passenger)
    flight = Field("flight", Flight)
    seat = Field("seat", Seat)
    baggage = Field("baggage", Baggage)
    payment = Field("payment", Payment)
    services = Field("services", field_type=Services)
    emergency_contact = Field("emergency_contact", EmergencyContact)
    notifications = Field("notifications", Notifications)

    def __init__(self, reservation_info):
        """Initialize a ReservationInfo object from reservation data.

        Args:
            reservation_info: Dictionary containing the complete flight
                reservation information.
        """
        self._data = reservation_info
```


Back to  [Articles](../articles.md)