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

Let's start by writing a simple function to read the COVID-19 dataset from a CSV file.
We'll use Python's built-in csv module to read the file and store each record as a list.
```python
from csv import reader

def read_csv(filename):
    with open(filename) as f:
        records = [ ]
        rows = reader(f)
        headers = next(rows)        # skipping headers
        for row in rows:
            records.append(row)
    return records
```
Here, `csv.reader()` reads the file one row at a time. We use `next(rows)` to retrieve 
and skip the header row, since our objective is to work with the actual data records.
Each subsequent row is appended to the records list.

Let's load the dataset and examine the resulting collection:
```commandline
>>> records = read_csv("data.csv")
>>> len(records)
583440
>>> records[0]
['04/01/20', 'AF', 'Afghanistan', 'EMR', '', '']
>>> records[1]
['04/01/20', 'DZ', 'Algeria', 'AFR', '', '']
>>> 
```
The dataset contains **583,440** records, with each record represented as a list of 
strings. This gives us our first data representation to measure, a list containing
individual lists.

In the following sections, we'll represent the same dataset using different 
Python data structures and compare their memory consumption.

Before comparing the memory consumption of different data structures, 
let's build a small, reusable memory profiler using Python's built-in `tracemalloc` 
module.

### Building a Memory Profiling Decorator
Before measuring the memory consumption of different data structures, 
let's build a small, reusable memory profiler.

Python provides the built-in `tracemalloc` module, which allows us to trace memory 
allocations made by Python code. We'll use it to measure the memory consumed while 
constructing each data representation.

Rather than repeating the profiling logic every time we want to measure memory usage,
we can encapsulate it in a class-based decorator.

```python
class Memory:
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