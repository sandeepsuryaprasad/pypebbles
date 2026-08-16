## Python Descriptors — Building a Reusable JSON Field Mapping Framework
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
<summary><strong>▸ Complete JSON Response</strong></summary>

<div markdown="1">

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

</details>

**Disclaimer:** The JSON data used in this article is for demonstration purposes. It does not represent actual production data, 
and any resemblance to real-world data is purely coincidental.

Let us re-use the code that we wrote in our previous article _From JSON Data to Python Objects_
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


Back to  [Articles](../articles.md)