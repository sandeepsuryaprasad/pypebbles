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
@Memory
def read_csv():
    with open("data.csv", encoding="utf-8-sig") as f:
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

### 4. List of Class instances

### 5. List of Class instances with `__slots__`

### 6. List of Data classes

### 7. List of Class NamedTuples