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
    "title": "Mr",
    "first_name": "John",
    "middle_name": "James",
    "last_name": "Doe",
    "date_of_birth": "1983-05-17",
    "gender": "MALE",
    "contact": {
      "email": "john.doe@example.com",
      "phone": "+1-123-555-0987",
      "alternate_phone": "+1-111-222-3333"
    },
    "address": {
      "street": "123 Example Avenue",
      "suite": "Suite 100",
      "city": "Sampleville",
      "state": "California",
      "state_code": "CA",
      "zip_code": "90000",
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
Response
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
│   ├── title
│   ├── first_name
│   ├── middle_name
│   ├── last_name
│   ├── date_of_birth
│   ├── gender
│   │
│   ├── contact
│   │   ├── email
│   │   ├── phone
│   │   └── alternate_phone
│   │
│   ├── address
│   │   ├── street
│   │   ├── suite
│   │   ├── city
│   │   ├── state
│   │   ├── state_code
│   │   ├── zip_code
│   │   └── country
│   │
│   └── frequent_flyer
│       ├── program
│       ├── membership_number
│       ├── status
│       └── miles_balance
│
├── flight
│   │
│   ├── airline
│   │   ├── code
│   │   ├── name
│   │   └── headquarters
│   │       ├── city
│   │       ├── state
│   │       └── state_code
│   │
│   ├── flight_number
│   ├── flight_status
│   │
│   ├── aircraft
│   │   ├── registration
│   │   ├── manufacturer
│   │   ├── model
│   │   └── configuration
│   │
│   ├── departure
│   │   ├── airport
│   │   │   ├── code
│   │   │   ├── name
│   │   │   ├── city
│   │   │   ├── state
│   │   │   ├── state_code
│   │   │   ├── country
│   │   │   └── terminal
│   │   │
│   │   ├── scheduled
│   │   │   ├── date
│   │   │   ├── time
│   │   │   └── timezone
│   │   │
│   │   ├── gate
│   │   └── boarding_time
│   │
│   ├── arrival
│   │   ├── airport
│   │   │   ├── code
│   │   │   ├── name
│   │   │   ├── city
│   │   │   ├── state
│   │   │   ├── state_code
│   │   │   ├── country
│   │   │   └── terminal
│   │   │
│   │   ├── scheduled
│   │   │   ├── date
│   │   │   ├── time
│   │   │   └── timezone
│   │   │
│   │   └── gate
│   │
│   ├── duration
│   │   ├── hours
│   │   └── minutes
│   │
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
│   │
│   ├── carry_on
│   │   ├── allowed_pieces
│   │   └── weight_limit
│   │       ├── value
│   │       └── unit
│   │
│   └── personal_item
│       ├── allowed
│       └── description
│
├── payment
│   ├── status
│   ├── method
│   │   ├── type
│   │   └── provider
│   │
│   └── fare
│       ├── base_fare
│       ├── taxes
│       ├── airport_fees
│       ├── service_fee
│       └── total
│
├── services [array]
│   │
│   ├── service
│   │   ├── code
│   │   ├── name
│   │   ├── description
│   │   └── status
│   │
│   └── ...
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
    │
    ├── email
    │   ├── enabled
    │   └── address
    │
    ├── sms
    │   ├── enabled
    │   └── phone
    │
    └── push
        └── enabled
```
</div>
</details>


Let us re-use the code that I wrote in previous article _From JSON Data to Python Objects_
with some minor modifications.
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

`Field` descriptor
The Field class is a descriptor that provides controlled attribute access to
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
        raise AttributeError(f"Cant set the value {value} to field {self.name}")
```
* `name` stores the corresponding key in the JSON object.
* `field_type` optionally specifies the Python class used to represent a 
nested JSON object.

when we say `flight = Field("flight", Flight)` the flight attribute maps to
the "flight" key in the JSON data and should be represented by a Flight object because
when "flight" key is accessed, the resulting value is one more nested JSON object.

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
* Attribute mapping - maps Python attributes to JSON keys.
* Nested object conversion - converts nested dictionaries into appropriate Python objects.
* Read-only access - prevents modification of the underlying JSON data.

This allows code to navigate a deeply nested JSON response using normal 
Python attribute access, while the descriptor handles the underlying dictionary
access and object creation transparently.

```commandline
>>> reservation.info.flight.departure.airport.city
'Austin'
>>> reservation.info.passenger.contact.email
'john.doe@example.com'
```
The following table summarizes the mapping between each JSON node, 
its corresponding Python attribute, and the Python types used to model 
nested JSON objects.

| JSON node           | Python attribute    | Mapped type        |
| ------------------- | ------------------- | ------------------ |
| `reservation`       | `reservation`       | `Reservation`      |
| `passenger`         | `passenger`         | `Passenger`        |
| `flight`            | `flight`            | `Flight`           |
| `seat`              | `seat`              | `Seat`             |
| `baggage`           | `baggage`           | `Baggage`          |
| `payment`           | `payment`           | `Payment`          |
| `services`          | `services`          | `Services`         |
| `emergency_contact` | `emergency_contact` | `EmergencyContact` |
| `notifications`     | `notifications`     | `Notifications`    |

### Implementing the Mapped Types

We will begin with the simpler JSON objects and gradually build the more 
complex types by composing them together. For nested JSON objects, 
the Field descriptor will be configured with the corresponding mapped type so
that the nested dictionary is automatically converted into an
instance of that type.

```python
class Reservation:
    confirmation_number = Field("confirmation_number")
    booking_status = Field("booking_status")
    booking_date = Field("booking_date")
    ticket_status = Field("ticket_status")
    currency = Field("currency")

    def __init__(self, reservation_info):
        self._data = reservation_info
```

The Reservation class represents the `reservation` JSON node. 
Each class attribute is a `Field` descriptor that maps a Python attribute to a
corresponding key in the underlying JSON data.

Let me create an instance of `FlightReservation`
```commandline
>>> reservation = FlightReservation()
>>> reservation.info
<__main__.ReservationInfo object at 0x104526c10>
```
when you say `reservation.info`, the `info` property returns an instance to
`ReservationInfo` class. Now you can access all the attributes of `ReservationInfo` class.
```commandline
>>> reservation.info.reservation
<__main__.Reservation object at 0x104542790>
>>> reservation.info.reservation.confirmation_number
'X7K9PQ'
>>> reservation.info.reservation.booking_status
'CONFIRMED'
>>> reservation.info.reservation.booking_date
'2026-08-10T14:32:18-05:00'
>>> reservation.info.reservation.ticket_status
'TICKETED'
>>> reservation.info.reservation.currency
'USD'
```
In the above code snippet, we are accessing attributes `confirmation_number`, `booking_status`
`booking_date`, `ticket_status` and `currency` of `Reservation` class.

Let us implement `Passenger` class,
```python
class Passenger:
    passenger_id = Field("passenger_id")
    title = Field("title")
    first_name = Field("first_name")
    middle_name = Field("middle_name")
    last_name = Field("last_name")
    date_of_birth = Field("date_of_birth")
    gender = Field("gender")
    contact = Field("contact", field_type=Contact)
    address = Field("address", field_type=Address)
    frequent_flyer = Field("frequent_flyer", field_type=FrequentFlyer)

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
>>> reservation.info.passenger.passenger_id
'PAX-104582'
>>> reservation.info.passenger.title
'Mr'
>>> reservation.info.passenger.first_name
'John'
>>> reservation.info.passenger.last_name
'Doe'
>>> reservation.info.passenger.date_of_birth
'1983-05-17'
>>> reservation.info.passenger.gender
'MALE'
```

But accessing `contact` or `address` or `frequent_flyer`, will not return the actual
value of corresponding node in JSON, but rather returns an instance of 
`Contact`, `Address` and `FrequentFlyer` class respectively. Because `contact` node
is a nested JSON object with `email`, `phone` and `alternate_phone`, which we have
abstracted in a different class `Contact`, similarly `address` and `frequent_flyer` nodes
of JSON response.

If the JSON node has a nested attribute, we are going pass the class reference to
`Field` descriptor through argument `field_type`, which then the descriptor returns
instance of the class that is passed.

```commandline
>>> reservation.info.passenger.contact
<__main__.Contact object at 0x104542850>
>>> reservation.info.passenger.address
<__main__.Address object at 0x104542820>
>>> reservation.info.passenger.frequent_flyer
<__main__.FrequentFlyer object at 0x1045427c0>
```

Let us implement `Contact`, `Address` and `FrequentFlyer` classes.
```python
class Contact:
    email = Field("email")
    phone = Field("phone")
    alternate_phone = Field("alternate_phone")

    def __init__(self, contact_info):
        self._data = contact_info
```
```python
class Address:
    street = Field("street")
    suite = Field("suite")
    city = Field("city")
    state = Field("state")
    state_code = Field("state_code")
    zip_code = Field("zip_code")
    country = Field("country")

    def __init__(self, address_info):
        self._data = address_info
```
```python
class FrequentFlyer:
    program = Field("program")
    membership_number = Field("membership_number")
    status = Field("status")
    miles_balance = Field("miles_balance")

    def __init__(self, frequent_flyer_info):
        self._data = frequent_flyer_info
```

You can access the nested `contact`, `address`, and `frequent_flyer` JSON nodes
through their corresponding Python attributes as shown below.

```commandline
>>> reservation.info.passenger.contact.email
'john.doe@example.com'
>>> reservation.info.passenger.contact.phone
'+1-123-555-0987'
>>> reservation.info.passenger.contact.alternate_phone
'+1-111-222-3333'
```
```commandline
>>> reservation.info.passenger.address.street
'123 Example Avenue'
>>> reservation.info.passenger.address.suite
'Suite 100'
>>> reservation.info.passenger.address.city
'Sampleville'
>>> reservation.info.passenger.address.state
'California'
```
```commandline
>>> reservation.info.passenger.frequent_flyer.program
'Spam Airlines dvantage'
>>> reservation.info.passenger.frequent_flyer.membership_number
'SA987654321'
>>> reservation.info.passenger.frequent_flyer.status
'Executive Platinum'
>>> reservation.info.passenger.frequent_flyer.miles_balance
184250
```

When you access `services` on `ReservationInfo` class, it returns a `list` of `dict`,
```commandline
>>> reservation.info.services
[{'code': 'MEAL', 'name': 'Premium Meal', 'description': 'Chicken and roasted vegetables', 'status': 'CONFIRMED'}, {'code': 'WIFI', 'name': 'Inflight Wi-Fi', 'description': 'High-speed internet access', 'status': 'CONFIRMED'}, {'code': 'LOUNGE', 'name': 'Elite Club', 'description': 'Airport lounge access', 'status': 'CONFIRMED'}]
>>> 
```

Now If you want to access the list of services that the passenger has opted for,
you should index the list.
```commandline
>>> reservation.info.services
[{'code': 'MEAL', 'name': 'Premium Meal', 'description': 'Chicken and roasted vegetables', 'status': 'CONFIRMED'}, {'code': 'WIFI', 'name': 'Inflight Wi-Fi', 'description': 'High-speed internet access', 'status': 'CONFIRMED'}, {'code': 'LOUNGE', 'name': 'Elite Club', 'description': 'Airport lounge access', 'status': 'CONFIRMED'}]
>>> 
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


Back to  [Articles](../articles.md)