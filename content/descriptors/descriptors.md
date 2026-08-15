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

**Disclaimer:** The JSON data used in this article is for demonstration purposes. It does not represent actual production data, 
and any resemblance to real-world data is purely coincidental.

```json
{
  "reservation": {
    "confirmation_number": "DEMO8X",
    "booking_status": "CONFIRMED",
    "booking_date": "2026-08-10T14:32:18-05:00",
    "ticket_status": "TICKETED",
    "currency": "USD"
  },
  "passenger": {
    "passenger_id": "PAX-DEMO001",
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
      "suite": "Demo Suite 100",
      "city": "Demoville",
      "state": "California",
      "state_code": "CA",
      "zip_code": "90000",
      "country": "United States"
    },
    "frequent_flyer": {
      "program": "Spam Airlines advantage",
      "membership_number": "SA987654321",
      "status": "Executive Platinum",
      "miles_balance": 184250
    }
  },
  "flight": {
    "airline": {
      "code": "SPM",
      "name": "Spam Airlines",
      "headquarters": {
        "city": "Austin",
        "state": "Texas",
        "state_code": "TX"
      }
    },
    "flight_number": "SPM1234",
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
      "base_fare": 485.00,
      "taxes": 72.75,
      "airport_fees": 18.40,
      "service_fee": 25.00,
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
      "suite": "Example Suite 777",
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

Back to  [Articles](../articles.md)