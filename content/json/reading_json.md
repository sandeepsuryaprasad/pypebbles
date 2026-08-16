## From JSON Data to Python Objects
**Last Updated:** August 2, 2026

Python makes it easy to work with JSON through its built-in `json` module. 
We can load a JSON document into Python and access its contents using dictionaries, 
lists, and standard indexing operations. While this approach works well for simple data 
structures, navigating deeply nested JSON using repeated dictionary lookups can quickly
become difficult to read and maintain.

In this article, we will take a different approach. Instead of exposing the raw JSON 
structure throughout our application, we will transform the data into a hierarchy of
Python objects, with each object responsible for representing a specific part of the data.

The objective is not merely to read JSON, but to design a Python interface that
makes complex structured data easier to work with.

### Simple JSON structure
`employees.json`
```json
[
    {
    "id": 1,
    "first_name": "David",
    "last_name": "Brown",
    "gender": "Male",
    "date_of_birth": "1983-08-26",
    "nationality": "United States"
  },
    {
    "id": 2,
    "first_name": "Laura",
    "last_name": "White",
    "gender": "Female",
    "date_of_birth": "1993-05-17",
    "nationality": "United States"
  }
]
```
For demonstration purpose, the above `json` file has a list of only two employee records. 
Each employee has fields, `id`, `first_name`, `last_name`, `gender`, `date_of_birth` and `nationality`. 

Let's design an object-oriented solution for reading and accessing data from the above JSON file.

`employee.py`
```python
from json import load
from pathlib import Path


class Employee:
    def __init__(self, emp_id):
        self.emp_id = emp_id
        self._path = self._json_file_path
        self._data = self._load_json_data
        self._info = None

    @property
    def _json_file_path(self):
        path = Path("employees.json")
        if not path.exists():
            raise FileNotFoundError(f"{path} does not exist")
        return path

    @property
    def _load_json_data(self):
        with open(self._path, "r") as json_file:
            json_object = load(json_file)
            for item in json_object:
                if item["id"] == self.emp_id:
                    return item
            raise ValueError(f"Invalid Employee ID {self.emp_id}")

    @property
    def info(self):
        if not self._info:
            self._info = EmployeeInfo(self._data)
        return self._info
```
```python
class EmployeeInfo:
    def __init__(self, employee_info):
        self.first_name = employee_info.get("first_name")
        self.last_name = employee_info.get("last_name")
        self.gender = employee_info.get("gender")
        self.date_of_birth = employee_info.get("date_of_birth")
        self.nationality = employee_info.get("nationality")
```
In the above `employee.py` file, we have two levels of abstraction, 
`Employee` and `EmployeeInfo` (both classes are in the same python module).

The `Employee` class encapsulates the logic for loading the JSON file and retrieving 
the details of the employee corresponding to the employee ID supplied during object 
instantiation.
Now let's access the actual employee data from JSON file. 

I am going to run python interpreter in interactive mode, 
```commandline
~$ python3 -i employee.py
>>> emp1 = Employee(1)    # Creating instance of first employee
>>> emp1.info.first_name
'David'
>>> emp1.info.last_name
'Brown'
>>> emp1.info.gender
'Male'
```

```commandline
>>> emp2 = Employee(2)    # Creating instance of second employee
>>> emp2.info.first_name
'Laura'
>>> emp2.info.last_name
'White'
>>> emp2.info.date_of_birth
'1993-05-17'
>>> emp2.info.nationality
'United States'
```
### Little more complicated JSON structure
`employees.json`

<details open>
<summary><strong>▸ Click here to expand/collapse complete JSON response</strong></summary>

<div markdown="1">

```json
[
  {
    "id": 1,
    "name": "Michael Anderson",
    "email": "michael.anderson@example.com",
    "address": {
      "street": "245 Oakwood Drive",
      "suite": "Suite 210",
      "city": "Austin",
      "state": "TX",
      "zipcode": "78701",
      "geo_location": {
        "lat": "30.2672",
        "lng": "-97.7431"
      }
    },
    "phone": "(000) 111-2222",
    "website": "www.example.dev",
    "company": {
      "name": "Spam Technologies",
      "industry": "Automobile"
    }
  },
  {
    "id": 2,
    "name": "Emily Johnson",
    "email": "emily.johnson@example.com",
    "address": {
      "street": "1187 Pine Street",
      "suite": "Apt. 5B",
      "city": "Seattle",
      "state": "WA",
      "zipcode": "98101",
      "geo_location": {
        "lat": "47.6062",
        "lng": "-122.3321"
      }
    },
    "phone": "(111) 333-4567",
    "website": "www.spam.com",
    "company": {
      "name": "Demo Software",
      "industry": "Software"
    }
  }
]
```


</div>
</details>

Now let's try the same solution that we used for our first example. I am going to keep `Employee` 
class as is and let's modify `EmployeeInfo` class to have the above JSON attributes.
```python
class EmployeeInfo:
    def __init__(self, employee_info):
        self.name = employee_info.get("name")
        self.email = employee_info.get("email")
        self.phone = employee_info.get("phone")
        self.website = employee_info.get("website")
        self.address = employee_info.get("address")
        self.company = employee_info.get("company")
        self.geo_location = employee_info.get("address").get("geo_location")
```

```commandline
>>> emp1 = Employee(1)
>>> emp1.info.name
'Michael Anderson'
>>> emp1.info.id
1
>>> emp1.info.email
'michael.anderson@example.com'
```
Now let's try to access `address` which returns one more JSON like object.
```commandline
>>> emp1.info.address
{'street': '245 Oakwood Drive', 'suite': 'Suite 210', 'city': 'Austin', 'state': 'TX', 'zipcode': '78701', 'geo_location': {'lat': '30.2672', 'lng': '-97.7431'}}
```
We got the correct output, which is the nested JSON object for the key `address`.
The nested JSON object has complete address information of an employee. Now if we want to access, let's say
`street` or `city` and if we did something like below, we would get `AttributeError`

```commandline
>>> emp1.info.address.street
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'dict' object has no attribute 'street'

>>> emp1.info.address.city
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'dict' object has no attribute 'city'
```
The problem becomes apparent when we attempt to access a nested attribute using 
an expression such as `emp1.info.address.street`. If `emp1.info.address` returns 
a dictionary, Python attempts to resolve `street` as an attribute of the `dict` 
object. Since the built-in `dict` type does not define an attribute named `street`, 
the attribute lookup fails with an `AttributeError`.
A dictionary provides access to its contents through key-based indexing, 
such as `address["street"]`, rather than attribute-based access using the dot 
operator. Therefore, we cannot directly use dot notation to traverse nested 
dictionary data. To support an expression such as `emp1.info.address.street`, 
the nested dictionary must first be represented by an object that exposes its keys 
as attributes. 

So the only way to make above code work is by doing something 
like this,
```commandline
>>> emp1.info.address.get("street")  # On dict object we are using `get` method
'245 Oakwood Drive'

>>> emp1.info.address["street"]     # Indexing syntax to access the key of a dict
'245 Oakwood Drive'
```
Similarly, when access `company`,
```commandline
>>> emp1.info.company
{'name': 'Spam Technologies', 'industry': 'Automobile'}
```
Now if we wanted to get `name` or `industry`, we would have to do something like this,
```commandline
>>> emp1.info.company.get("name")
'Spam Technologies'
>>> emp1.info.company.get("industry")
'Automobile'
```
This is really akward! and doesn't compose well. 

The most elegant solution should look something like this,
```commandline
>>> emp1.info.address.street
'245 Oakwood Drive'
>>> emp1.info.address.suite
'Suite 210'
>>> emp1.info.address.zipcode
'78701'
>>> 
>>> emp1.info.company.name
'Spam Technologies'
>>> emp1.info.company.industry
'Automobile' 
```
Let us take a look at the JSON file `employees.json`. In the above JSON , 
each employee has two more nested JSON  like objects `address` and `company`. 

In order to have complete object oriented approach to access the attributes of 
`address` and `company` let's introduce two more levels of abstraction for the 
above scenario.  We will create two separate classes for `Address` and `Company`.

```python
class Address:
    def __init__(self, address_info):
        self.street = address_info.get("street")
        self.suite = address_info.get("suite")
        self.city = address_info.get("city")
        self.state = address_info.get("state")
        self.zipcode = address_info.get("zipcode")
        self.geo_location = address_info.get("geo_location")
```
```python
class Company:
    def __init__(self, company_info):
        self.name = company_info.get("name")
        self.industry = company_info.get("industry")
```
Now let's change `EmployeeInfo` class implementation, 
```python
class EmployeeInfo:
    def __init__(self, employee_info):
        self.name = employee_info.get("name")
        self.email = employee_info.get("email")
        self.phone = employee_info.get("phone")
        self.website = employee_info.get("website")
        self.address = Address(employee_info.get("address"))
        self.company = Company(employee_info.get("company"))
```
Here is the magic,

```commandline
>>> emp1 = Employee(1)
>>> emp1.info.address.street    # accessing address information
'245 Oakwood Drive'
>>> emp1.info.address.suite
'Suite 210'
>>> emp1.info.address.city
'Austin'
>>> emp1.info.address.zipcode
'78701'
>>> 
>>> emp1.info.company.name  # accessing company information
'Spam Technologies'
>>> emp1.info.company.industry
'Automobile'
```
```commandline
>>> emp2 = Employee(2)
>>> emp2.info.address.street
'1187 Pine Street'
>>> emp2.info.address.suite
'Apt. 5B'
>>> emp2.info.address.city
'Seattle'
>>> emp2.info.address.zipcode
'98101'
>>> 
>>> emp2.info.company.name
'Demo Software'
>>> emp2.info.company.industry
'Software'
```
Here, `emp1.info.address` resolves to an instance of the `Address` class. 
When we evaluate `emp1.info.address.street`, Python performs attribute lookup 
for `street` on that `Address` instance and returns the corresponding instance 
attribute. This allows the nested address data to be accessed through standard 
object attribute notation rather than dictionary key-based access.

But we still have problem when we access `emp1.info.address.geo_location`. When `geo_location` 
attribute is looked-up on `address` object, again a dictionary is returned, and we have to access
that dictionary either using `get` or through indexing.

So we can solve this by creating one more layer of Abstraction, `Location`
```python
class Location:
    def __init__(self, location_info):
        self.lat = location_info.get("lat")
        self.lng = location_info.get("lng")
```
Now let's add interface to `Location` class in `Address`

```python
class Address:
    def __init__(self, address_info):
        self.street = address_info.get("street")
        self.suite = address_info.get("suite")
        self.city = address_info.get("city")
        self.state = address_info.get("state")
        self.zipcode = address_info.get("zipcode")
        self.geo_location = Location(address_info.get("geo_location"))
```
Now we can access `lat` and `lng` attributes of `Location` though `geo_location` in `Address` 
```commandline
>>> emp1.info.address.geo_location.lat
'30.2672'
>>> emp1.info.address.geo_location.lng
'-97.7431'
```
By introducing multiple layers of abstraction for nested JSON objects, 
we create a solution that is easier to read, maintain, and extend. 
Encapsulating the underlying JSON structure within dedicated classes results in cleaner, 
more modular code and provides a simple, intuitive interface for accessing nested data.

The final solution may look something like this,
```python
from json import load
from pathlib import Path


class Employee:
    """Represent an employee and provide access to employee information.

    The class loads employee data from `employees.json` based on the
    employee ID supplied during object creation. Employee information
    can be accessed through the `info` property.

    Args:
        emp_id: Unique identifier of the employee.
    """
    def __init__(self, emp_id):
        self.emp_id = emp_id
        self._path = self._json_file_path
        self._data = self._load_json_data
        self._info = None

    @property
    def _json_file_path(self):
        """Return the path to the employee JSON file.
        Returns:
            The path to `employees.json`.

        Raises:
            FileNotFoundError: If `employees.json` does not exist.
        """
        path = Path("employees.json")
        if not path.exists():
            raise FileNotFoundError(f"{path} does not exist")
        return path

    @property
    def _load_json_data(self):
        """Load and return data for the specified employee.
        Searches `employees.json` for an employee whose ID matches
        the ID supplied during object creation.

        Returns:
            A dictionary containing the employee's data, or an empty
            dictionary if no employee with the specified ID is found.
        """
        with open(self._path, "r") as json_file:
            json_object = load(json_file)
            for item in json_object:
                if item["id"] == self.emp_id:
                    return item
            raise ValueError(f"Invalid Employee ID {self.emp_id}")

    @property
    def info(self):
        """Return the employee information.
        Creates an `EmployeeInfo` instance lazily on first access and reuses 
        the same instance for subsequent accesses.

        Returns:
            An `EmployeeInfo` instance containing the employee's data.
        """
        if not self._info:
            self._info = EmployeeInfo(self._data)
        return self._info
```
```python
class EmployeeInfo:
    """Represent employee information as a structured Python object.

    Encapsulates the employee's personal, contact, address, and company
    information by converting the corresponding JSON data into strongly
    structured Python objects.

    Args:
        employee_info: Dictionary containing the employee information.
    """

    def __init__(self, employee_info):
        """Initialize an EmployeeInfo instance from employee data.

        Args:
            employee_info: Dictionary containing employee details and
            nested address and company information.
        """
        self.name = employee_info.get("name")
        self.email = employee_info.get("email")
        self.phone = employee_info.get("phone")
        self.website = employee_info.get("website")
        self.address = Address(employee_info.get("address"))
        self.company = Company(employee_info.get("company"))
```
```python
class Address:
    """Represent an employee's address information.

    Encapsulates address details and the associated geographical location
    represented by a :class:`Location` object.
    Args:
        address_info: Dictionary containing address details and
        geographical location information.
    """
    def __init__(self, address_info):
        self.street = address_info.get("street")
        self.suite = address_info.get("suite")
        self.city = address_info.get("city")
        self.state = address_info.get("state")
        self.zipcode = address_info.get("zipcode")
        self.geo_location = Location(address_info.get("geo_location"))
```
```python
class Company:
    """Represent an employee's company information.

    Encapsulates company-related details associated with an employee.

    Args:
        company_info: Dictionary containing company information.
    """
    def __init__(self, company_info):
        """Initialize a Company instance from company data.
        Args:
            company_info: Dictionary containing company details.
        """
        self.name = company_info.get("name")
        self.industry = company_info.get("industry")
```
```python
class Location:
    """Represent geographical location information.
    Encapsulates the latitude and longitude associated with an address.

    Args:
        location_info: Dictionary containing geographical coordinates.
    """
    def __init__(self, location_info):
        """Initialize a Location instance from geographical data.

        Args:
            location_info: Dictionary containing latitude and longitude
            values.
        """
        self.lat = location_info.get("lat")
        self.lng = location_info.get("lng")
```

In this article, we explored how a nested JSON structure can be transformed into a 
clean, object-oriented representation using Python. Instead of exposing dictionaries 
and requiring callers to navigate the JSON structure using keys and indexing, 
we introduced a hierarchy of Python objects that provides a more expressive and 
intuitive interface. By separating the data into focused classes such as 
`EmployeeInfo`, `Address`, `Location`, and `Company`, each class takes responsibility
for representing a specific part of the underlying data. This keeps the implementation
modular while allowing the client code to interact with the data through familiar 
dot notation

As applications grow and data structures become more complex, thoughtful
abstraction can make the difference between code that merely works and code that 
remains readable, maintainable, reusable, and extensible over time.

Back to  [Articles](../articles.md)