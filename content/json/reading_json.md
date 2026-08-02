## Reading JSON File Using Object-Oriented Approach
**Last Updated:** August 2, 2026

In this article, we will learn how to read a `json` file using object oriented approach. 
Let's consider the below JSON file,

`employees.json`
```json
[
    {
    "id": "1",
    "first_name": "David",
    "last_name": "Brown",
    "gender": "Male",
    "date_of_birth": "1983-08-26",
    "nationality": "United States"
  },
    {
    "id": "2",
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
    def __init__(self, emp_id: str):
        self.emp_id = str(emp_id)
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
>>>
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
>>> 
```

Back to  [Articles](../articles.md)