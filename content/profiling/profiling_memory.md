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

Let's write a function that reads data from csv file.
```python
from csv import reader

def read_csv():
    with open("data.csv") as f:
        records = [ ]
        rows = reader(f)
        headers = next(rows)        # skipping headers
        for row in rows:
            records.append(row)
    return records
```
```commandline
>>> records = read_csv()
>>> len(csv_data)
583440
>>> records[0]
['04/01/20', 'AF', 'Afghanistan', 'EMR', '', '']
>>> records[1]
['04/01/20', 'DZ', 'Algeria', 'AFR', '', '']
>>> 
```

