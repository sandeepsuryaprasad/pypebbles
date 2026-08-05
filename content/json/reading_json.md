## Reading JSON File Using Object-Oriented Approach
**Last Updated:** August 2, 2026

In this article, we will learn how to read a `json` file using object oriented approach. 
Let's consider the below JSON file,

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
        """Returns a JSON file path object"""
        path = Path("employees.json")
        if not path.exists():
            raise FileNotFoundError(f"{path} does not exist")
        return path

    @property
    def _load_json_data(self):
        """
        Loads the JSON file and returns the employee data for the employee corresponding 
        to the employee ID supplied during object instantiation.

        Returns an empty dictionary if the employee id does not exist in the JSON file
        """
        with open(self._path, "r") as json_file:
            json_object = load(json_file)
            for item in json_object:
                if item["id"] == self.emp_id:
                    return item
            return {}

    @property
    def info(self):
        if not self._info:
            self._info = EmployeeInfo(self._data)
        return self._info
```
```python
class EmployeeInfo:
    def __init__(self, data):
        self._data = data

    @property
    def first_name(self):
        return self._data.get("first_name", "")

    @property
    def last_name(self):
        return self._data.get("last_name", "")

    @property
    def gender(self):
        return self._data.get("gender", "")

    @property
    def date_of_birth(self):
        return self._data.get("date_of_birth", "")

    @property
    def nationality(self):
        return self._data.get("nationality", "")
```
In the above `employee.py` file, we have two levels of abstraction, 
`Employee` and `EmployeeInfo` (both classes are in the same python module).

The `Employee` class encapsulates the logic for loading the JSON file and retrieving 
the details of the employee corresponding to the employee ID supplied during object 
instantiation.

The `EmployeeInfo` class has a bunch of `property` or `getters`, corresponding to employee 
attributes in JSON record (e.g. `first_name`, `last_name`, `gender` etc).

Now let's access the actual employee data from JSON file. 
I am going to run python interpreter in interactive mode, 
```commandline
~$ python3 -i employee.py
>>> emp1 = Employee("1")    # Creating instance of first employee
>>> emp1.info.first_name
'David'
>>> emp1.info.last_name
'Brown'
>>> emp1.info.gender
'Male'
```

```commandline
>>> emp2 = Employee("2")    # Creating instance of second employee
>>> emp2.info.first_name
'Laura'
>>> emp2.info.last_name
'White'
>>> emp2.info.date_of_birth
'1993-05-17'
>>> emp2.info.nationality
'United States'
```
### Alternate Approach: Dynamic Attribute Access Using `__getattr__` (hack)
The `EmployeeInfo` class, has five different `property` or `getter` methods, one for each
employee info filed. If you feel that it's kind of boilerplate code with many `getter` methods,
you can use the magic of `__getattr__` method to optimize the code.
```python
class EmployeeInfo:
    def __init__(self, data):
        self._data = data

    def __getattr__(self, name):
        if name not in self._data.keys():
            raise AttributeError(f"{self.__class__.__name__} object has no attribute {name}")
        return self._data.get(name)
```
In the above code, when we try to access an attribute on `EmployeeInfo` object by saying,

```commandline
>>> emp1 = Employee("1")
>>> emp1.info.first_name    # we are doing an attribute look-up on EmployeeInfo object
```
Here the name of the attribute that we are trying to do a look-up is `first_name`, which
obviously does not exist on `EmployeeInfo` object itself (The only instance attribute that
`EmployeeInfo` has is `_data`). Since we are trying to access a missing attribute on
`EmployeeInfo` object, `__getattr__` method gets automatically called for missing attributes and
the name the missing attribute is passed-in as a string to `__getattr__` method and that string 
is collected by parameter `name`.

In `__getattr__` method we are validating if the attribute for which we are trying to get the value,
in this case `first_name` is present as key of the dictionary `self._data`. If it is present
then we are doing a dictionary look-up and returning the value of key `first_name`.

The same process happens for other employee attributes are well.

### Little more complicated JSON structure
`employees.json`

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
Now let's try the same solution that we used for our first example. I am going to keep `Employee` 
class and `EmployeeInfo` (we will use the one which has `__getattr__` implemented) class as is for now. 
Let's run the code and see how it works for the new JSON object.

```commandline
>>> emp1 = Employee(1)
>>> emp1.info.name
'Michael Anderson'
>>> emp1.info.id
1
>>> emp1.info.email
'michael.anderson@example.com'
```
Now let's try to access `address` which returns one more JSON like object or inner dictionary.
```commandline
>>> emp1.info.address
{'street': '245 Oakwood Drive', 'suite': 'Suite 210', 'city': 'Austin', 'state': 'TX', 'zipcode': '78701', 'geo_location': {'lat': '30.2672', 'lng': '-97.7431'}}
```
We got the correct output, which is the inner dictionary or nested dictionary for the key `address`.
The nested dictionary has complete address information of an employee. Now if we want to access, let's say
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
Here is the problem, when we execute `emp1.info.address.street`, `emp1.info.address` returns a dictionary object.
On a dictionary object, we are using dot operator to access `street` which is wrong. Because `dict` 
object does not have an attribute `street`. We cannot be using `dot` operator to access the contents
of the dictionary. 

So the only way to make above code work is by doing something like this,
```commandline
>>> emp1.info.address.get("street")  # on dict object we are using a metod `get`
'245 Oakwood Drive'

>>> emp1.info.address["street"]     # Indexing syntax to access the key of a dict
'245 Oakwood Drive'
```
Similarly, when we try to access `company`,
```commandline
>>> emp1.info.company
{'name': 'Spam Technologies', 'industry': 'Automobile'}
```
Now if we wanted to get `name` or `industry`, we would do something like this,
```commandline
>>> emp1.info.company.get("name")
'Spam Technologies'
>>> emp1.info.company.get("industry")
'Automobile'
```
This is really akward! 

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
Let us take a look at the JSON file `employees.json`. In the above JSON , each employee has 
two more JSON  like objects `address` and `company`. 

In order to have complete object oriented approach to access the attributes of `address` and `company`
let's introduce two more levels of abstraction for the above scenario. 
We will create two separate classes for `address` and `company`.

```python
class Address:
    def __init__(self, data):
        self._data = data

    def __getattr__(self, name):
        if name not in self._data.keys():
            raise AttributeError(f"{self.__class__.__name__} has no attribute {name}")
        return self._data.get(name)

class Company:
    def __init__(self, data):
        self._data = data

    def __getattr__(self, name):
        if name not in self._data.keys():
            raise AttributeError(f"{self.__class__.__name__} has no attribute {name}")
        return self._data.get(name)
```
Now let's change `EmployeeInfo` class implementation, 
```python
class EmployeeInfo:
    def __init__(self, data):
        self._data = data

    def __getattr__(self, name):
        if name not in self._data.keys():
            raise AttributeError(f"{self.__class__.__name__} has no attribute {name}")
        if name == "address":
            return Address(self._data.get(name))
        if name == "company":
            return Company(self._data.get(name))
        return self._data.get(name)
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
Here `emp1.info.address` returns `Address` object, when we say `emp1.info.address.street`, we 
are trying to look-up for an attribute `street` on `Address` object. Since `__getattr__` method
is implemented in `Address` class, the name of the attribute, in this case `street` is passed-in to
`__getattr__` method in string format. In `__getattr__` method, we are going to pull out the key `street`
from internal dictionary `self._dict` of `Address` class.

But we still have problem when we access `emp1.info.address.geo_location`. When `geo_location` 
attribute is looked-up on `address` object, again a dictionary is returned, and we have to access
that dictionary either using `get` or through indexing.

So Here is what we can do to solve that,
```python
class Address:
    def __init__(self, data):
        self._data = data

    def __getattr__(self, name):
        if name == "geo_location":
            return Location(self._data.get(name))
        if name not in self._data.keys():
            raise AttributeError(f"{self.__class__.__name__} has not attribute {name}")
        return self._data.get(name, "")

class Location(Address):
    """Subclass of Address"""
    pass
```

```commandline
>>> emp1.info.address.geo_location.lat
'30.2672'
>>> emp1.info.address.geo_location.lng
'-97.7431'
```
In this way by creating multiple layers of abstraction for nested JSON objects, we can make our solution
more readable, maintainable and scalable.

The final solution may look something like this, 
```python
class EmployeeInfo:
    def __init__(self, data):
        self._data = data

    def __getattr__(self, name):
        if name not in self._data.keys():
            raise AttributeError(f"{self.__class__.__name__} has not attribute {name}")
        if name == "address":
            return Address(self._data.get(name))
        if name == "company":
            return Company(self._data.get(name))
        return self._data.get(name)

class Address:
    def __init__(self, data):
        self._data = data

    def __getattr__(self, name):
        if name not in self._data.keys():
            raise AttributeError(f"{self.__class__.__name__} has not attribute {name}")
        if name == "geo_location":
            return Location(self._data.get(name))
        return self._data.get(name, "")

class Company:
    def __init__(self, data):
        self._data = data

    def __getattr__(self, name):
        if name not in self._data.keys():
            raise AttributeError(f"{self.__class__.__name__} has not attribute {name}")
        return self._data.get(name, "")

class Location(Address):
    """Subclass of Address"""
    pass
```

Back to  [Articles](../articles.md)