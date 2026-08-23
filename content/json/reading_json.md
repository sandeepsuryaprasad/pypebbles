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
For the purpose demonstration, let's consider the above `json` file that has a list of 
only two employee records. Each employee record has fields, 
`id`, `first_name`, `last_name`, `gender`, `date_of_birth` and `nationality`. 

Let's design an object-oriented solution for reading and accessing data from the above JSON file.

`employee.py`
```python
from json import load
from pathlib import Path


class Employees:
    """Provide sequence-style access to employee information.

    Loads employee records from ``data.json`` and converts each record into
    an :class:`Employee` object during object initialization. The resulting
    collection can be accessed through the ``employees`` property or using
    standard sequence operations such as indexing and ``len()``.

    Attributes:
        _path: Path to the JSON file containing employee records.
        _data: Raw employee records loaded from the JSON file.
        _employees: Collection of :class:`Employee` objects created from
            the raw employee records.
    """

    def __init__(self):
        """Initialize the Employees collection.

        Resolves the JSON file path, loads the employee records, and
        converts the records into :class:`Employee` objects.
        """
        self._path = self._json_file_path
        self._data = self._load_json_data
        self._employees = self.employees

    @property
    def _json_file_path(self) -> Path:
        """Return the path to the employee JSON file.

        Returns:
            Path: Path to ``data.json``.

        Raises:
            FileNotFoundError: If ``data.json`` does not exist.
        """
        path = Path("data.json")
        if not path.exists():
            raise FileNotFoundError(f"{path} does not exist")
        return path

    @property
    def _load_json_data(self) -> list[dict]:
        """Load and deserialize employee data from the JSON file.

        Returns:
            list[dict]: A list of dictionaries representing employee
                records.
        """
        with open(self._path, "r") as json_file:
            return load(json_file)

    @property
    def employees(self) -> list[Employee]:
        """Return all employees as Employee objects.

        Converts each raw employee dictionary into an :class:`Employee`
        object, providing a structured Python representation of the
        underlying JSON data.

        Returns:
            list[Employee]: A list of Employee objects.
        """
        self._employees = [Employee(employee) for employee in self._data]
        return self._employees

    def __getitem__(self, index):
        """Return the employee at the specified index.

        Provides sequence-style indexed access to the employee collection.

        Args:
            index: Zero-based index of the employee to retrieve.

        Returns:
            Employee: The employee at the specified index.
        """
        return self._employees[index]

    def __len__(self):
        """Return the number of employees in the collection.

        Returns:
            int: Number of Employee objects in the collection.
        """
        return len(self._employees)
```
Technically, the class performs three main operations:

* Resolves and validates the JSON file path through the `_json_file_path` property. 
It raises a `FileNotFoundError` if the expected file is unavailable.

* Loads and deserializes the JSON data through `_load_json_data`. 
The JSON document is converted into native Python objects, 
resulting in a `list[dict]` where each dictionary represents an `Employee` record.

* The `employees` property wraps each raw employee dictionary in an `Employee` object, 
abstracting the underlying JSON representation behind a structured Python interface.

* Sequence-style access, by implementing `__getitem__`, the Employees class 
supports indexed access

```python
class Employee:
    """Represent an employee using structured employee information.

    Encapsulates the employee's personal information and exposes the
    corresponding JSON fields as Python attributes.

    Attributes:
        first_name: Employee's first name.
        last_name: Employee's last name.
        gender: Employee's gender.
        date_of_birth: Employee's date of birth.
        nationality: Employee's nationality.
    """

    def __init__(self, employee_info):
        """Initialize an Employee object from employee data.

        Args:
            employee_info: Dictionary containing the employee's personal
                information.
        """
        self.first_name = employee_info["first_name"]
        self.last_name = employee_info["last_name"]
        self.gender = employee_info["gender"]
        self.date_of_birth = employee_info["date_of_birth"]
        self.nationality = employee_info["nationality"]
```

Next, let's access the employee data loaded from the JSON file. 
To demonstrate this interactively, we will launch the Python interpreter in interactive 
mode and inspect the resulting objects and their attributes.

```commandline
~$ python3 -i employee.py
```

```python
>>> employees = Employees() # creating instance of `Employees` class
>>> employees
<__main__.Employees object at 0x100d16a30>
>>> employees.employees     # accessing `employees` property on `Employees` object
[<__main__.Employee object at 0x100d33760>, <__main__.Employee object at 0x100d336d0>]
```
```python
>>> employees.employees[0]  # indexing the employees list
<__main__.Employee object at 0x100d33580>
>>> employees.employees[0].first_name
'David'
>>> employees.employees[0].last_name
'Brown'
```
```python
>>> employees.employees[1].first_name
'Laura'
>>> employees.employees[1].last_name
'White'
```
Since the `Employees` class implements the `__getitem__` special method, 
its instances support sequence-style indexing and can be iterated over using 
Python's iteration protocol.
```python
>>> employees[0]
<__main__.Employee object at 0x10040aa30>   # instance of `Employee` class
>>> employees[1]
<__main__.Employee object at 0x10040aee0>   # instance of `Employee` class
```
```python
>>> employees[0].first_name
'David'
>>> employees[0].last_name
'Brown'
```
```python
>>> employees[1].first_name
'Laura'
>>> employees[1].last_name
'White'
```
We can iterate over `employees` object itself.
```python
>>> for employee in employees:
...     print(employee.first_name, employee.last_name)
... 
David Brown
Laura White
>>> 
```
You can ask for length of `employees` object. 
```python
>>> len(employees)
2
```

### Nested JSON structure
`employees.json`
```json
[
  {
    "id": 1,
    "name": "Michael Anderson",
    "email": "michael.anderson@example.com",
    "website": "www.example.dev",
    "address": {
      "city": "Austin",
      "state": "TX",
      "country": "United States",
      "geo_location": {"lat": "30.2672", "lng": "-97.7431"}
    },
    "skills": [
      {"name": "Python", "level": "Advanced"},
      {"name": "C", "level": "Advanced"},
      {"name": "C++", "level": "Intermediate"}
    ]
  },
  {
    "id": 2,
    "name": "Emily Johnson",
    "email": "emily.johnson@example.com",
    "website": "www.spam.com",
    "address": {
      "city": "Seattle",
      "state": "WA",
      "country": "United States",
      "geo_location": {"lat": "47.6062", "lng": "-122.3321"}
    },
    "skills": [
      {"name": "Ruby", "level": "Intermediate"},
      {"name": "Rust", "level": "Advanced"}
    ]
  }
]
```
Below is the JSON structure for the above response
```commandline
Employees [array]
│
├── Employee [object]
│   ├── id
│   ├── name
│   ├── email
│   ├── website
│   │
│   ├── address [object]
│   │   ├── city
│   │   ├── state
│   │   ├── country
│   │   │
│   │   └── geo_location [object]
│   │       ├── lat
│   │       └── lng
│   │
│   └── skills [array]
│       │
│       ├── Skill [object]
│       │   ├── name
│       │   └── level
│       │
│       ├── Skill [object]
│       │   ├── name
│       │   └── level
│       │
│       └── Skill [object]
│           ├── name
│           └── level
│
└── Employee [object]
    ├── id
    ├── name
    ├── email
    ├── website
    │
    ├── address [object]
    │   ├── city
    │   ├── state
    │   ├── country
    │   │
    │   └── geo_location [object]
    │       ├── lat
    │       └── lng
    │
    └── skills [array]
        │
        ├── Skill [object]
        │   ├── name
        │   └── level
        │
        └── Skill [object]
            ├── name
            └── level
```
Few things to be noted in the above JSON structure, 
* `address` is a `dict` object which has one more `dict` object `geo_location`
* `skills` is a `list` object, each item of the list is a `dict` object

Now let's try the same solution that we used for our first example. 
I am going to keep `Employees` class as it is and let's modify `Employee` class to 
have the above JSON attributes.
```python
class Employee:
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
        self.emp_id = employee_info["id"]
        self.name = employee_info["name"]
        self.email = employee_info["email"]
        self.website = employee_info["website"]
        self.address = employee_info["address"]
        self.skills = employee_info["skills"]
```
```python
>>> employees = Employees()
>>> employees.employees
[<__main__.Employee object at 0x108eeb7f0>, <__main__.Employee object at 0x108ecec10>]
```
```python
>>> employees.employees[0]
<__main__.Employee object at 0x108eeb850>
```
```python
>>> employees.employees[0].emp_id
1
>>> employees.employees[0].name
'Michael Anderson'
```
```python
>>> employees.employees[1].emp_id
2
>>> employees.employees[1].email
'emily.johnson@example.com'
```
However, when we access `address` it returns one more `dict` object.
```python
>>> employees.employees[0].address
{'city': 'Austin', 'state': 'TX', 'country': 'United States', 'geo_location': {'lat': '30.2672', 'lng': '-97.7431'}}
```
The expression correctly returns the nested JSON object associated with the address key,
which contains the employee's complete address information. However, attempting to 
access a nested field such as city or state using attribute notation at this stage will 
result in an AttributeError, because the returned value is still a standard Python 
dictionary rather than an Address object.

```python
>>> employees.employees[0].address.city
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'dict' object has no attribute 'city'
```
The problem becomes apparent when we attempt to access a nested attribute using 
an expression such as `employees.employees[0].address.city`. If `employees.employees[0].address` 
returns a dictionary, Python attempts to resolve `city` as an attribute of the `dict` 
object. Since the built-in `dict` type does not define an attribute named `city`, 
the attribute lookup fails with an `AttributeError`.
A dictionary provides access to its contents through key-based indexing, 
such as `address["city"]`, rather than attribute-based access using the dot 
operator. Therefore, we cannot directly use dot notation to traverse nested 
dictionary data. To support an expression such as `employees.employees[0].address.city`, 
the nested dictionary must first be represented by an object that exposes its keys 
as attributes. 

So the only way to make above code work is by doing something like this,
```python
>>> employees.employees[0].address["city"]  # Indexing Syntax to access the key of the dict
'Austin'
>>> employees.employees[0].address.get("city")     # On dict object we are using `get` method
'Austin'
```
To access the `geo_location` details, you must first navigate through the nested 
`address` object and then access the required fields within the `geo_location` structure.

```python
>>> employees.employees[0].address["geo_location"]
{'lat': '30.2672', 'lng': '-97.7431'}
```
To access `lat` and `lng` values,
```python
>>> employees.employees[0].address["geo_location"]["lat"]
'30.2672'
>>> 
>>> employees.employees[0].address["geo_location"]["lng"]
'-97.7431'
```
A similar challenge arises with the `skills` attribute. Accessing skills on an `Employee` 
object returns a `list` containing the skills associated with that employee. 
Since an employee can have multiple skills, each element in the list is represented
as a separate dictionary containing the corresponding skill details.
```python
>>> employees.employees[0].skills
[{'name': 'Python', 'level': 'Advanced'}, {'name': 'C', 'level': 'Advanced'}, {'name': 'C++', 'level': 'Intermediate'}]
>>> 
>>> employees.employees[1].skills
[{'name': 'Ruby', 'level': 'Intermediate'}, {'name': 'Rust', 'level': 'Advanced'}]
```
Suppose if we wanted to access the actual values, we would do something like below,
```python
>>> employees.employees[0].skills[0]        # first employee skills
{'name': 'Python', 'level': 'Advanced'}
>>> employees.employees[0].skills[0]["name"]
'Python'
>>> employees.employees[0].skills[0]["level"]
'Advanced'
>>> 
>>> employees.employees[0].skills[1]
{'name': 'C', 'level': 'Advanced'}
>>> 
>>> employees.employees[0].skills[1]["name"]
'C'
>>> 
>>> employees.employees[0].skills[1]["level"]
'Advanced'
>>> 
```
```python
>>> employees.employees[1].skills   # second employee skills
[{'name': 'Ruby', 'level': 'Intermediate'}, {'name': 'Rust', 'level': 'Advanced'}]
>>> 
>>> employees.employees[1].skills[0]    
{'name': 'Ruby', 'level': 'Intermediate'}
>>> 
>>> employees.employees[1].skills[1]
{'name': 'Rust', 'level': 'Advanced'}
>>> 
>>> employees.employees[1].skills[0]["name"]
'Ruby'
>>> 
>>> employees.employees[1].skills[0]["level"]
'Intermediate'
```
The current implementation feels awkward because it exposes the underlying dictionary 
structure and doesn't compose well.

In order to have complete object oriented approach to access the attributes of 
`address` `geo_location` and `skills`, let's introduce few more levels
of abstraction for the above scenario `Address`, `Location`, and `Skills`.
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
        self.lat = location_info["lat"]
        self.lng = location_info["lng"]
```
```python
class Address:
    """Represent an employee's address information.

    Encapsulates the address and geographical information associated with
    an employee and exposes the corresponding JSON fields as Python
    attributes.

    Attributes:
        street: Street address.
        suite: Apartment, suite, or unit information.
        city: City associated with the address.
        state: State or administrative region.
        zipcode: Postal or ZIP code.
        geo_location: Geographical coordinates associated with the address.
    """

    def __init__(self, address_info):
        """Initialize an Address object from address data.

        Args:
            address_info: Dictionary containing the employee's address
                and geographical information.
        """
        self.street = address_info["street"]
        self.suite = address_info["suite"]
        self.city = address_info["city"]
        self.state = address_info["state"]
        self.zipcode = address_info["zipcode"]
        self.geo_location = Location(address_info["geo_location"])
```
```python
class Skills:
    """Represent a collection of skills associated with an employee.

    Encapsulates the list of skill records and converts each raw skill
    dictionary into a structured :class:`Skill` object. The collection
    supports indexed access to individual skills.

    Attributes:
        skills: List of :class:`Skill` objects representing the employee's
            skills.
    """

    class Skill:
        """Represent an individual employee skill.

        Attributes:
            name: Name of the skill.
            level: Proficiency level associated with the skill.
        """

        def __init__(self, skill_info):
            """Initialize a Skill object from skill data.

            Args:
                skill_info: Dictionary containing the skill name and
                    proficiency level.
            """
            self.name = skill_info["name"]
            self.level = skill_info["level"]

    def __init__(self, skills):
        """Initialize a Skills collection from skill data.

        Args:
            skills: List of dictionaries containing employee skill
                information.
        """
        self.skills = [self.Skill(skill) for skill in skills]

    def __getitem__(self, index):
        """Return the skill at the specified index.

        Args:
            index: Zero-based index of the skill to retrieve.

        Returns:
            Skill: The skill object at the specified index.
        """
        if index > len(self.skills) - 1:
            raise IndexError(f"Skill index must be less than {len(self.skills)}")
        return self.skills[index]
```
Now, let's modify the Employee class so that the address and skills attributes are 
represented by corresponding Address and Skills objects, rather than exposing the 
underlying JSON structures directly. 
```python
class Employee:
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
        self.emp_id = employee_info["id"]
        self.name = employee_info["name"]
        self.email = employee_info["email"]
        self.website = employee_info["website"]
        self.address = Address(employee_info["address"])
        self.skills = Skills(employee_info["skills"])
```
This is where the abstraction pays off. The underlying JSON structure is encapsulated
behind a clean, object-oriented interface.
```python
>>> employees.employees[0].address.city
'Austin'
>>> employees.employees[0].address.state
'TX'
>>> employees.employees[0].address.country
'United States'
>>> 
>>> employees.employees[1].address.city
'Seattle'
>>> employees.employees[1].address.state
'WA'
>>> employees.employees[1].address.country
'United States'
```
Here, `employees.employees[0].address` resolves to an instance of the `Address` class. 
When we evaluate `employees.employees[0].address.city`, Python performs attribute lookup 
for `city` on that `Address` instance and returns the corresponding instance 
attribute. This allows the nested address data to be accessed through standard 
object attribute notation rather than dictionary key-based access.
```python
>>> employees.employees[0].address.geo_location.lat
'30.2672'
>>> employees.employees[0].address.geo_location.lng
'-97.7431'
```
```python
>>> employees.employees[1].address.geo_location.lat
'47.6062'
>>> employees.employees[1].address.geo_location.lng
'-122.3321'
```
```python
>>> employees.employees[0].skills[0].name
'Python'
>>> employees.employees[0].skills[0].level
'Advanced'
>>> 
>>> employees.employees[0].skills[1].name
'C'
>>> 
>>> employees.employees[0].skills[2].name
'C++'
```
```python
>>> employees.employees[1].skills[0].name
'Ruby'
>>> 
>>> employees.employees[1].skills[1].name
'Rust'
>>> 
>>> employees.employees[1].skills[0].level
'Intermediate'
>>> 
>>> employees.employees[1].skills[1].level
'Advanced'
```
Since the `Skills` class implements the `__getitem__` special method, 
its instances support sequence-style indexing and can be iterated over using 
Python's iteration protocol.
```python
>>> for skill in employees.employees[0].skills:
...     print(f"{skill.name}, {skill.level}")
... 
Python, Advanced
C, Advanced
C++, Intermediate
```
```python
>>> for skill in employees.employees[1].skills:
...     print(f"{skill.name}, {skill.level}")
... 
Ruby, Intermediate
Rust, Advanced
```
### Final Thoughts
By introducing multiple layers of abstraction for nested JSON objects, 
we create a solution that is easier to read, maintain, and extend. 
Encapsulating the underlying JSON structure within dedicated classes results in cleaner, 
more modular code and provides a simple, intuitive interface for accessing nested data.

In this article, we explored how a nested JSON structure can be transformed into a 
clean, object-oriented representation using Python. Instead of exposing dictionaries 
and requiring callers to navigate the JSON structure using keys and indexing, 
we introduced a hierarchy of Python objects that provides a more expressive and 
intuitive interface. By separating the data into focused classes such as 
`Employee`, `Address`, `Location` and `Skills`, each class takes responsibility
for representing a specific part of the underlying data. This keeps the implementation
modular while allowing the client code to interact with the data through familiar 
dot notation

As applications grow and data structures become more complex, thoughtful
abstraction can make the difference between code that merely works and code that 
remains readable, maintainable, reusable, and extensible over time.

Back to  [Articles](../articles.md)