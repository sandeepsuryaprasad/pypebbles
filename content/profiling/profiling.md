[Articles](../articles.md) \|  [Previous](../logging/logging.md)

## Profiling in Python

In this article, we will build a reusable Python performance profiler that combines 
`cProfile` for function-level execution analysis with `perf_counter` for measuring 
wall-clock elapsed time. The profiler will initially be implemented as a context manager 
and then extended using function decorators and class decorators, allowing the same 
profiling functionality to be applied to individual functions or entire classes with 
minimal changes to application code. 

The examples use API calls to demonstrate profiling in a realistic scenario, 
where the majority of the elapsed time may be spent waiting for a remote server 
rather than executing Python code locally. This distinction is important when analyzing
application performance because CPU execution time and end-to-end response time represent
different aspects of system performance.

The objective is not simply to measure how fast a piece of code runs, but to build a 
reusable profiling utility that can help identify performance characteristics and 
potential bottlenecks in real-world Python applications.

### Building a Reusable Profiler

Instead of placing `perf_counter` and `cProfile` calls directly inside every function
that needs to be measured, we can encapsulate the profiling logic in a reusable class.

The Profiler class acts as a context manager, allowing profiling to be enabled 
automatically when entering a with block and disabled when leaving it. 
This keeps the profiling logic separate from the application code being measured.

The class combines two complementary approaches:

* `perf_counter` measures the total wall-clock time taken by an operation.
* `cProfile` collects detailed information about function calls made during that operation.
* `pstats.Stats` provides an interface for sorting, displaying, and exporting the 
collected profiling statistics.

This design allows the same profiler to be reused for API calls, file operations, 
database operations, or any other Python code where execution performance needs to 
be investigated.

The following implementation provides the basic profiling functionality. 
Each method has a specific responsibility: starting the profiler, stopping it, calculating 
elapsed time, displaying profiling statistics, and exporting the collected 
statistics for further analysis.

```python
class Profiler:
    """Measure execution time and collect Python profiling statistics.

    The Profiler class combines ``time.perf_counter()`` with
    ``cProfile`` to measure wall-clock elapsed time and collect
    detailed function-call statistics.

    The class is designed to be used as a context manager::

        with Profiler() as profiler:
            some_function()

        print(profiler.elapsed_time)
        profiler.print_stats()

    Attributes:
        _profile: The ``cProfile.Profile`` instance used to collect
            profiling data.
        _stats: The ``pstats.Stats`` instance created from the collected
            profiling data.
        _start: The value of ``perf_counter()`` recorded when profiling
            starts.
        _end: The value of ``perf_counter()`` recorded when profiling
            ends.
    """

    def __init__(self):
        """Initialize a Profiler instance.

        The actual profiling session is created when the context manager
        is entered.
        """
        self._profile = None
        self._stats = None
        self._start = 0.0
        self._end = 0.0

    def __enter__(self):
        """Start the profiling session.

        Creates a ``cProfile.Profile`` instance, enables profiling, and
        records the starting wall-clock time using ``perf_counter()``.

        Returns:
            Profiler: The current Profiler instance.
        """
        self._profile = Profile(builtins=False)
        self._profile.enable()
        self._start = perf_counter()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        """Stop the profiling session and collect statistics.

        Records the ending wall-clock time, disables ``cProfile``,
        and creates a ``pstats.Stats`` object from the collected
        profiling data.

        Args:
            exc_type: Exception type if an exception was raised inside
                the context manager; otherwise ``None``.
            exc_val: Exception instance if an exception was raised;
                otherwise ``None``.
            exc_tb: Traceback object associated with the exception;
                otherwise ``None``.
        """
        self._end = perf_counter()
        self._profile.disable()
        self._stats = Stats(self._profile)

    def print_stats(self, limit=10, sort_key=SortKey.CUMULATIVE):
        """Display collected profiling statistics.

        The statistics are stripped of directory information, sorted
        using the specified sort key, and printed to the console.

        Args:
            limit: Maximum number of profiling entries to display.
                Defaults to ``10``.
            sort_key: The ``pstats.SortKey`` used to order the results.
                Defaults to ``SortKey.CUMULATIVE``.
        """
        self._stats.strip_dirs().sort_stats(sort_key)
        self._stats.print_stats(limit)

    @property
    def elapsed_time(self):
        """Return the total wall-clock elapsed time in seconds.

        Returns:
            float: The elapsed time between entering and exiting the
                profiling context, rounded to three decimal places.
        """
        return round(self._end - self._start, 3)

    def dump_stats(self, filename):
        """Save profiling statistics to a file.

        The generated statistics file can be loaded later using
        ``pstats.Stats`` or other tools that understand cProfile
        statistics files.

        Args:
            filename: Path of the file where the profiling statistics
                should be saved.
        """
        self._stats.dump_stats(filename)
```

[Articles](../articles.md) \|  [Previous](../logging/logging.md)