## Memory Profiling

When working with large datasets, the way data is represented in memory can have a 
significant impact on the memory consumption of a Python application. 
Two different data structures may represent exactly the same information while requiring 
very different amounts of memory. To explore this, this article uses a real-world COVID-19 
dataset containing approximately half a million records collected from countries around the world.

### Large CSV file
You can download the entire dataset  [HERE](https://data.who.int/dashboards/covid19/data)

The size of the csv file around 19 MB and it has 583440 rows of data in it.
```commandline
Date_reported,Country_code,Country,WHO_region,New_cases,New_deaths
04/01/20,AF,Afghanistan,EMR,,
04/01/20,DZ,Algeria,AFR,,
04/01/20,AL,Albania,EUR,,
04/01/20,AI,Anguilla,AMR,,
04/01/20,AS,American Samoa,WPR,0,0
...
```

Each record contains information such as the reporting date, country, WHO region, number 
of new cases, and number of new deaths. Rather than simply loading the CSV file and 
measuring its memory consumption, we will represent the same dataset using several 
different Python data structures, including **tuples**, **dictionaries**, **namedtuples**, 
**class instances**, class instances with **slots** attribute defined and **dataclass** objects. 
We will then use Python's built-in **tracemalloc** module to measure and compare the memory 
allocated by each representation.

The objective is to learn how tracemalloc works, but to understand how 
our choice of data structure can affect the memory footprint of a Python application 
when working with large collections of objects.

We will begin by examining the dataset and creating different representations of the
same records. We will then introduce tracemalloc to trace memory allocations and
build a reusable memory-profiling approach that can be applied to functions and larger 
pieces of application code.

By the end of the article, we will have a clearer understanding of how Python data 
structures affect memory usage and how memory profiling can replace assumptions with 
measurable evidence when making design decisions.

### Building a Memory Profiling Decorator
Before measuring the memory consumption of different data structures, 
let's build a small, reusable memory profiler.

Python provides the built-in `tracemalloc` module, which allows us to trace memory 
allocations made by Python code. We'll use it to measure the memory consumed while 
constructing each data representation.

Rather than repeating the profiling logic every time we want to measure memory usage,
we can encapsulate it in a class-based decorator.

```python
class MemoryProfiler:
    def __init__(self, func):
        """Initialize the memory profiler with the function to be profiled."""
        self.trace = tracemalloc
        self.func = func

    def start(self):
        """Start tracing memory allocations if tracing is inactive."""
        if not self.trace.is_tracing():
            self.trace.start()

    def stop(self):
        """Stop tracing memory allocations if tracing is active."""
        if self.trace.is_tracing():
            self.trace.stop()

    def memory_usage(self):
        """Display the current and peak memory usage in megabytes."""
        current, peak = self.trace.get_traced_memory()
        print(f"Current Memory Usage: {current / 1024 / 1024:.2f} MB")
        print(f"Peak Memory Usage:    {peak / 1024 / 1024:.2f} MB")

    def __call__(self, *args, **kwargs):
        """Profile the memory usage of the decorated function."""
        self.start()
        try:
            result = self.func(*args, **kwargs)
            print(f"Total Number of Rows: {len(result)}")
            self.memory_usage()
            return result
        finally:
            self.stop()
```
The Memory class implements the decorator protocol through its `__call__` method. 
When applied to a function, it starts memory tracing, executes the function, 
reports the current and peak memory usage, and stops tracing.

The decorator also returns the original function's result, allowing us to continue '
working with the data structure after measuring its memory consumption.

We'll use this decorator throughout the experiment to profile the different Python 
data representations.

In the following sections, we'll represent the same dataset using different 
Python data structures and compare their memory consumption.

### 1. List of Lists
Let's begin with the simplest representation. We'll store the CSV data in a Python 
list, where each record is represented by an individual list containing the values 
from that CSV row.

Let's start by writing a simple function to read the COVID-19 dataset from a CSV file
and decorate the function with class implementation of `MemoryProfiler` decorator.
We'll use Python's built-in csv module to read the file and store each record as a
list.
```python
from csv import reader

@MemoryProfiler
def read_csv(filename):
    with open(filename, encoding="utf-8-sig") as f:
        records = [ ]
        rows = reader(f)
        headers = next(rows)        # skipping headers
        for row in rows:
            records.append(row)
    return records
```
Let's load the dataset and examine the results:
```commandline
>>> records = read_csv("data.csv")
Total Number of Rows: 583440
Current Memory Usage: 184.45 MB
Peak Memory Usage:    184.60 MB
>>> 
>>> records[0]
['04/01/20', 'AF', 'Afghanistan', 'EMR', '', '']
>>> records[1]
['04/01/20', 'DZ', 'Algeria', 'AFR', '', '']
>>> 
```
The dataset contains **583,440** records, with each record is represented as a list of 
strings.

### Understanding the Memory Consumption
In this representation, the dataset is stored as a list of lists. The outer list holds 
583,440 individual record lists, and each inner list contains six items corresponding 
to the columns in the CSV file.

For this dataset, `tracemalloc` reports a peak memory usage of **184.60 MB** while constructing 
the list of lists. This gives us a baseline against which we can compare the memory
consumption of the other data structures.

### 2. List of Dictionaries
Next, let's represent the same dataset using a list of dictionaries. 
Instead of storing each CSV record as a list of values, we'll represent it as a
dictionary containing key-value pairs, where the CSV column names serve as dictionary 
keys and the corresponding row values serve as dictionary values.

```python
from csv import reader

@MemoryProfiler
def read_csv(filename):
    with open(filename, encoding="utf-8-sig") as f:
        records = [ ]
        rows = reader(f)
        headers = next(rows)        # skipping headers
        for row in rows:
            _row = { header: value for header, value in zip(headers, row)}
            records.append(_row)
    return records
```
```commandline
>>> records = read_csv("data.csv")
Total Number of Rows: 583440
Current Memory Usage: 269.05 MB
Peak Memory Usage:    269.20 MB
>>> records[0]
{'Date_reported': '04/01/20', 'Country_code': 'AF', 'Country': 'Afghanistan', 'WHO_region': 'EMR', 'New_cases': '', 'New_deaths': ''}
>>> records[1]
{'Date_reported': '04/01/20', 'Country_code': 'DZ', 'Country': 'Algeria', 'WHO_region': 'AFR', 'New_cases': '', 'New_deaths': ''}
>>> 
```
After converting the dataset into a list of dictionaries, we can see that its peak memory 
consumption has increased to **269.18 MB**, compared with **184.60 MB** for the list of
lists. That's an increase of approximately **45.8%**.

This increase is primarily due to the additional memory overhead introduced by Python 
dictionaries. Each record is now represented by a dictionary that maintains key-value 
pairs instead of a simple list of values.

Every dictionary stores references to its keys and values, along with additional internal 
storage required to manage and retrieve those entries. Since we create 583,440 individual 
dictionaries, this overhead accumulates across the dataset.

Although this representation makes the data easier to understand and access using 
meaningful column names, it comes at a higher memory cost.

### 3. List of Tuples
Next, let's represent each CSV record as a tuple instead of a list. Like lists, 
tuples store an ordered collection of values and allow access through numeric indexes.

The key difference is that tuples are immutable, meaning their elements cannot be 
modified after creation. Since our dataset is primarily used for storing and analysing 
records, tuples provide a suitable alternative for evaluating memory consumption.
```python
@MemoryProfiler
def read_csv(filename):
    with open(filename, encoding="utf-8-sig") as f:
        records = [ ]
        rows = reader(f)
        headers = next(rows)        # skipping headers
        for row in rows:
            records.append(tuple(row))
    return records
```
```commandline
>>> records = read_csv("data.csv")
Total Number of Rows: 583440
Current Memory Usage: 171.13 MB
Peak Memory Usage:    171.28 MB
>>> records[0]
('04/01/20', 'AF', 'Afghanistan', 'EMR', '', '')
>>> 
>>> records[1]
('04/01/20', 'DZ', 'Algeria', 'AFR', '', '')
```
After converting the dataset into a list of tuples, the peak memory consumption drops to
**171.28 MB**, compared with **184.60 MB** for the list of lists. That's a reduction of **7.2%**
compare to lists.

The reduction in memory consumption is primarily due to the difference in how Python 
manages the internal storage of lists and tuples. Lists are mutable and support dynamic 
resizing, so Python over-allocates their internal storage to accommodate future additions 
through operations such as `append`. Tuples, being immutable, have a fixed size after
creation and do not require this additional capacity for subsequent modifications.

### 4. List of Class Instances
Next, let's represent each CSV record as an instance of a custom Python class named `Covid`.
Instead of storing values in a list, tuple, or dictionary, we'll define attributes for the
fields in each record.

```python
class Covid:
    def __init__(self, date, country_code, country, who_region, new_cases, new_deaths):
        self.date = date
        self.country_code = country_code
        self.country = country
        self.who_region = who_region
        self.new_cases = new_cases
        self.new_deaths = new_deaths

@MemoryProfiler
def read_csv(filename):
    with open(filename, encoding="utf-8-sig") as f:
        records = [ ]
        rows = reader(f)
        headers = next(rows)        # skipping headers
        for row in rows:
            covid = Covid(*row)
            records.append(covid)
    return records
```
```commandline
>>> records = read_csv("data.csv")
Total Number of Rows: 583440
Current Memory Usage: 184.48 MB
Peak Memory Usage:    184.63 MB
>>> 
>>> records[0]
<__main__.Covid object at 0x1050156a0>
>>> 
>>> records[1]
<__main__.Covid object at 0x104fb74d0>
>>> 
>>> records[0].__dict__
{'date': '04/01/20', 'country_code': 'AF', 'country': 'Afghanistan', 'who_region': 'EMR', 'new_cases': '', 'new_deaths': ''}
>>> 
>>> records[1].__dict__
{'date': '04/01/20', 'country_code': 'DZ', 'country': 'Algeria', 'who_region': 'AFR', 'new_cases': '', 'new_deaths': ''}
>>> 
>>> records[0].date
'04/01/20'
>>> 
>>> records[0].country_code
'AF'
```
When each CSV record is represented as an instance of the `Covid` class, the peak memory 
consumption is approximately **184 MB**. Although we are using a custom class to represent 
each record, Python stores the instance attributes in an internal instance dictionary 
`__dict__`.

An interesting observation emerges when we compare this result with the list-of-dictionaries 
representation. In that case, each record is stored as a regular Python dictionary,
resulting in a peak memory consumption of approximately **269 MB**. However,
storing the same data as class instances requires only around **184 MB**.

This difference is largely due to Python's memory optimisation for instance dictionaries.
When multiple instances of the same class are created, 
Python can use **key-sharing dictionaries** (PEP 412). Since all instances share the same 
set of attribute names, Python can share the dictionary keys and their associated 
metadata rather than maintaining a separate copy of the keys for every instance. 
Each instance then maintains its own attribute values.

Regular dictionaries, on the other hand, generally maintain their own key storage. 
Consequently, representing records as instances of a common class can be more
memory-efficient than representing them as independent dictionaries, even though both 
approaches provide key-based access to their values.

### 5. List of Class instances with slots
In the previous representation, each `Covid` instance maintained an instance dictionary 
`__dict__` to store its attributes. Python dictionaries provide flexibility for 
dynamically adding and modifying attributes, but they introduce additional memory overhead.

To reduce this overhead, we can use Python's `__slots__` mechanism. By declaring the 
permitted attribute names at the class level, Python doesn't store the attribute values in
an instance dictionary, but the values are stored inside a tuple like immutable object.

```python
class Covid:
    __slots__ = ("date", "country_code", "country", "who_region", "new_cases", "new_deaths") 
    def __init__(self, date, country_code, country, who_region, new_cases, new_deaths):
        self.date = date
        self.country_code = country_code
        self.country = country
        self.who_region = who_region
        self.new_cases = new_cases
        self.new_deaths = new_deaths

@MemoryProfiler
def read_csv():
    with open("data.csv", encoding="utf-8-sig") as f:
        records = [ ]
        rows = reader(f)
        headers = next(rows)        # skipping headers
        for row in rows:
            covid = Covid(*row)
            records.append(covid)
    return records
```
```commandline
>>> records = read_csv("data.csv")
Total Number of Rows: 583440
Current Memory Usage: 162.22 MB
Peak Memory Usage:    162.37 MB
>>> 
>>> records[0]
<__main__.Covid object at 0x101b953f0>
>>> 
>>> records[0].__dict__
Traceback (most recent call last):
  File "<python-input-3>", line 1, in <module>
    records[0].__dict__
AttributeError: 'Covid' object has no attribute '__dict__'. Did you mean: '__dir__'?
>>> 
>>> hasattr(records[0], "__dict__")
False
>>> 
```
With `__slots__`, the peak memory consumption drops to **162.37 MB**, compared with 
approximately **184.23 MB** without slots, representing a reduction of **11.86%**.

This demonstrates how `__slots__` can improve memory efficiency when creating large
numbers of instances with a fixed set of attributes.

### 6. List of Dataclass Instances
Python's `dataclasses` module provides a convenient way to define classes primarily 
intended to store data. Instead of writing an explicit `__init__` method, we can use
the `@dataclass` decorator to generate it automatically based on the declared fields.

Let's represent each CSV record as an instance of the `Covid` dataclass.

```python
from dataclasses import dataclass

@dataclass
class Covid:
    data: str
    country_code: str
    country: str
    who_region: str
    new_cases: str
    new_deaths: str

@MemoryProfiler
def read_csv():
    with open("data.csv", encoding="utf-8-sig") as f:
        records = [ ]
        rows = reader(f)
        headers = next(rows)        # skipping headers
        for row in rows:
            covid = Covid(*row)
            records.append(covid)
    return records
```
```commandline
>>> records = read_csv("data.csv")
Total Number of Rows: 583440
Current Memory Usage: 184.48 MB
Peak Memory Usage:    184.63 MB
>>> 
```
The resulting peak memory consumption is **184.63 MB**, which is almost identical to
the memory consumed by instances of our regular `Covid` class without `__slots__`.

This is expected because, by default, dataclasses use a regular instance dictionary 
`__dict__` to store their attributes. The `@dataclass` decorator simplifies class 
definition and object initialization, but it does not inherently reduce the memory 
footprint of instances.

Therefore, while dataclasses offer a cleaner and more concise way to define data-oriented 
classes, they do not provide the memory savings achieved through `__slots__` by default.

### 7. List of Dataclass Instances with slots
Python's dataclasses module provides built-in support for `__slots__` through the 
`slots=True` parameter. This feature, introduced in Python 3.10, allows us to create 
dataclass instances with slot-based attribute storage without explicitly declaring 
`__slots__` in the class definition.

When `slots=True` is specified, Python generates slot descriptors for the declared 
dataclass fields. As a result, instances do not require a separate instance dictionary 
`__dict__` to store their attributes.

For our dataset of 583,440 records, the peak memory consumption drops to **162.37 MB**, 
matching the result obtained with our manually implemented `__slots__` class.

This demonstrates that we can achieve the same memory efficiency as a manually 
implemented slotted class while retaining the convenience of dataclasses, including
automatically generated initialization and other useful methods.

```python
from dataclasses import dataclass

@dataclass(slots=True)
class Covid:
    date: str
    country_code: str
    country: str
    who_region: str
    new_cases: str
    new_deaths: str
```
```commandline
>>> records = read_csv("data.csv")
Total Number of Rows: 583440
Current Memory Usage: 162.22 MB
Peak Memory Usage:    162.37 MB
>>>
```
### 8. List of NamedTuples
So far, we have represented each CSV record using lists, dictionaries, tuples, and 
custom class instances. Another useful alternative is namedtuple, available in Python's 
collections module.

A `namedtuple` is a tuple subclass that allows its elements to be accessed using 
meaningful attribute names instead of numeric indexes. Like regular tuples, 
namedtuples are immutable, but they provide a more descriptive and readable way to
work with structured records.

Let's define a Covid namedtuple with the six fields from our dataset and create one
instance for each CSV record.

```python
from collections import namedtuple

Covid = namedtuple("Covid", ("date", "country_code", "country", "who_region", "new_cases", "new_deaths"))

@MemoryProfiler
def read_csv(filename):
    with open(filename, encoding="utf-8-sig") as f:
        records = [ ]
        rows = reader(f)
        headers = next(rows)        # skipping headers
        for row in rows:
            covid = Covid(*row)
            records.append(covid)
    return records
```
```commandline
>>> records = read_csv("data.csv")
Total Number of Rows: 583440
Current Memory Usage: 175.58 MB
Peak Memory Usage:    175.73 MB
>>> 
>>> records[0]
Covid(date='04/01/20', country_code='AF', country='Afghanistan', who_region='EMR', new_cases='', new_deaths='')
>>> 
>>> records[1]
Covid(date='04/01/20', country_code='DZ', country='Algeria', who_region='AFR', new_cases='', new_deaths='')
>>> 
>>> records[0][0]   # indexing
'04/01/20'
>>> 
>>> records[0].date  # attribute access
'04/01/20' 
```
The measured peak memory consumption is **175.73 MB**. This is higher than the **171.28 MB**
consumed by the list of regular tuples. The additional memory is associated with the
namedtuple instance structure, which provides attribute-based access while retaining 
tuple behaviour.

However, namedtuples still consume less memory than ordinary class instances with
an instance dictionary or a list of dictionaries. They offer a useful balance between
memory efficiency and readable attribute access.

### Memory Consumption Summary
| No. | Data Representation | Current Memory (MB) | Peak Memory (MB) |
|---:|---|---:|---:|
| 1 | List of Lists | 184.45 | 184.60 |
| 2 | List of Dictionaries | 269.05 | 269.20 |
| 3 | List of Tuples | 171.13 | 171.28 |
| 4 | List of Class Instances | 184.48 | 184.63 |
| 5 | List of Class Instances with Slots | 162.22 | 162.37 |
| 6 | List of Dataclass Instances | 184.48 | 184.63 |
| 7 | List of Dataclass Instances with Slots | 162.22 | 162.37 |
| 8 | List of NamedTuples | 175.58 | 175.73 |
