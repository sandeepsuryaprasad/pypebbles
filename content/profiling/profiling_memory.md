## Memory Profiling

When working with large datasets, the way data is represented in memory can have a 
significant impact on the memory consumption of a Python application. 
Two different data structures may represent exactly the same information while requiring 
very different amounts of memory. To explore this, this article uses a real-world COVID-19 
dataset containing approximately half a million records collected from countries around the world.

Each record contains information such as the reporting date, country, WHO region, number 
of new cases, and number of new deaths. Rather than simply loading the CSV file and 
measuring its memory consumption, we will represent the same dataset using several 
different Python data structures, including tuples, dictionaries, namedtuples, class instances,
class instances with slots attribute defined and dataclass objects. 
We will then use Python's built-in tracemalloc module to measure and compare the memory 
allocated by each representation.

The objective is not only to learn how tracemalloc works, but also to understand how 
our choice of data structure can affect the memory footprint of a Python application 
when working with large collections of objects.

We will begin by examining the dataset and creating different representations of the
same records. We will then introduce tracemalloc to trace memory allocations and
build a reusable memory-profiling approach that can be applied to functions and larger 
pieces of application code.

By the end of the article, we will have a clearer understanding of how Python data 
structures affect memory usage and how memory profiling can replace assumptions with 
measurable evidence when making design decisions.

You can download the dataset  [HERE](https://data.who.int/dashboards/covid19/data)