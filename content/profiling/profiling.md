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

    The profiling results can be sorted and limited when they are
    displayed. The class is designed to be used as a context manager::

        with Profiler(limit=10) as profiler:
            some_function()

        print(profiler.elapsed_time)
        profiler.print_stats()

    Args:
        limit: Maximum number of profiling entries to display.
        sort_key: The ``pstats.SortKey`` used to sort the profiling
            statistics. Defaults to ``SortKey.CUMULATIVE``.

    Attributes:
        _profile: The ``cProfile.Profile`` instance used to collect
            profiling data.
        _stats: The ``pstats.Stats`` instance created from the collected
            profiling data.
        _start: The wall-clock time recorded when profiling starts.
        _end: The wall-clock time recorded when profiling ends.
        _limit: Maximum number of profiling entries to display.
        _sort_key: Sorting criterion used when displaying profiling
            statistics.
    """

    def __init__(self, limit=10, sort_key=SortKey.CUMULATIVE):
        """Initialize a Profiler instance.

        Args:
            limit: Maximum number of profiling entries to display.
            sort_key: The ``pstats.SortKey`` used to sort the profiling
                statistics. Defaults to ``SortKey.CUMULATIVE``.
        """
        self._profile = None
        self._stats = None
        self._start = 0.0
        self._end = 0.0
        self._limit = limit
        self._sort_key = sort_key

    def __enter__(self):
        """Start the profiling session.

        Creates a ``cProfile.Profile`` instance, enables profiling,
        and records the starting wall-clock time using
        ``perf_counter()``.

        Returns:
            Profiler: The current Profiler instance.
        """
        self._profile = Profile(builtins=False)
        self._profile.enable()
        self._start = perf_counter()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        """Stop profiling and collect the profiling statistics.

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

    def print_stats(self):
        """Display the collected profiling statistics.

        The statistics are stripped of directory information, sorted
        using the configured sort key, and limited to the configured
        number of entries.
        """
        self._stats.strip_dirs().sort_stats(self._sort_key)
        self._stats.print_stats(self._limit)

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
        ``pstats.Stats`` or other tools that support cProfile
        statistics files.

        Args:
            filename: Path of the file where the profiling statistics
                should be saved.
        """
        self._stats.dump_stats(filename)
```

### Using the Profiler to profile API Calls

Now that we have implemented the Profiler class, let's see how it can be used to measure 
the performance of real-world operations.

To keep the example practical, we will use API requests rather than artificial functions
that perform simple calculations. API calls are particularly useful for demonstrating 
profiling because their execution time can be influenced by several factors, including 
network latency, server response time, and the processing performed by the HTTP client.


The following examples use different endpoints and response scenarios to demonstrate 
how the profiler behaves with API requests that complete at different speeds. 
Some requests return immediately, while others intentionally introduce a delay in the
server response.

For each API request, we can measure the total wall-clock elapsed time and, 
when required, inspect the function-call statistics collected by cProfile.

Let's start by applying the profiler explicitly to each API test. 
Consider the `test_delayed_users` test method, which invokes an HTTP `GET` 
request against a delayed API endpoint. We will use the `Profiler` class to measure 
the request's wall-clock execution time and analyze the underlying function-call 
activity captured by `cProfile`.

```python
from os import environ
from httpx import get
from pytest import fixture

headers = {
    "X-Reqres-Env": "prod",
    "x-api-key": environ["X_API_KEY"],
}

@fixture(scope="module")
def client():
    with Client() as _client:
        yield _client

def test_delayed_users(client):
    response = client.get("https://reqres.in/api/users?delay=2", headers=headers)
    assert response.status_code == 200
```
Before introducing any profiling , we will execute the `test_delayed_users` 
test independently using `pytest`
```commandline
~$ pytest -vs profiler.py::test_delayed_users
====================================== test session starts ==============================
platform darwin -- Python 3.9.6, pytest-7.4.4, pluggy-1.3.0 -- /Library/Developer/CommandLineTools/usr/bin/python3
cachedir: .pytest_cache
rootdir: /Users/sandeepsuryaprasad/Documents/articles/profiler
plugins: anyio-4.12.1, instafail-0.5.0, trio-0.8.0, mock-3.12.0
collected 1 item                                                                                                                                                                                        

profiler.py::test_delayed_users PASSED
====================================== 1 passed in 2.85s =============================== 
```
The test is passed. Now let's start by profiling the `test_delayed_users` test.

```python
def test_delayed_users(client):
    with Profiler() as profiler:
        response = client.get("https://reqres.in/api/users?delay=2", headers=headers)
        assert response.status_code == 200
    print(f"Elapsed Time: {profiler.elapsed_time}")
```
```commandline
~$ pytest -vs profiler.py::test_delayed_users
============================= test session starts ======================================
platform darwin -- Python 3.9.6, pytest-7.4.4, pluggy-1.3.0 -- /Library/Developer/CommandLineTools/usr/bin/python3
cachedir: .pytest_cache
rootdir: /Users/sandeepsuryaprasad/Documents/articles/profiler
plugins: anyio-4.12.1, instafail-0.5.0, trio-0.8.0, mock-3.12.0
collected 1 item                                                                                                                                                                                        

profiler.py::test_delayed_users :Elapsed Time: 2.675 secs
PASSED
=========================== 1 passed in 2.78s =========================================
```
Once again the test is passed, but this time the Elapsed Time is printed in the console
which is `2.675 secs`.

### Eliminating Profiling Code from Existing Tests
The context-manager approach works well when we are writing new code or when we
have complete control over the code being measured. However, consider a project in
which hundreds of tests have already been implemented. Adding a `with` `Profiler(...)` block
to every test would require modifying the existing test code and would introduce
profiling-specific logic into the tests themselves.

Ideally, we should be able to enable profiling without changing the implementation 
of the functions or tests that we want to measure.

That's where decorators come into picture. Python decorators provide an elegant solution 
to this problem. A decorator allows us to wrap an existing function with additional 
behavior without modifying the function's implementation.

We can therefore move the profiling logic out of the test and into a reusable `@profile` 
decorator. The test remains focused solely on its original purpose, 
while the decorator transparently handles starting the profiler, measuring 
execution time, collecting profiling statistics, and reporting the results.

Let's see how we can implement a reusable function decorator for this purpose.

Consider below tests that validates the response of code different end points,
```python
from os import environ
from httpx import get
from pytest import fixture

headers = {
    "X-Reqres-Env": "prod",
    "x-api-key": environ["X_API_KEY"],
}

@fixture(scope="module")
def client():
    with Client() as _client:
        yield _client

        
def test_single_user(client):
    response = client.get("https://reqres.in/api/users/2", headers=headers)
    assert response.status_code == 200

def test_user_not_found(client):
    response = client.get("https://reqres.in/api/users/23", headers=headers)
    assert response.status_code == 404

def test_list_users(client):
    response = client.get("https://reqres.in/api/users?page=2", headers=headers)
    assert response.status_code == 200

def test_resources(client):
    response = client.get("https://reqres.in/api/users?page=2", headers=headers)
    assert response.status_code == 200

def test_delayed_users(client):
    response = client.get("https://reqres.in/api/users?delay=2", headers=headers)
    assert response.status_code == 200

def test_more_delayed_users(client):
    response = client.get("https://reqres.in/api/users?delay=3", headers=headers)
    assert response.status_code == 200
```

We now have a set of existing API tests that exercise different endpoints and scenarios. 
These tests are already implemented and their primary responsibility is to validate the 
expected API behavior.

Our next objective is to profile these tests without adding profiling logic directly 
into each test function. Adding a `with` `Profiler(...)` block to every test would introduce 
repetitive instrumentation code and would mix performance-measurement concerns with test
logic.

Let's now implement a `profile` decorator that uses our `Profiler` class to measure and 
report the execution characteristics of these tests.

For the initial implementation, we will use a function decorator and explicitly decorate
each test function that we want to profile. This approach allows us to introduce 
profiling with minimal changes to the existing test code while keeping the profiling 
logic separate from the test implementation.

Later in this article, we will refactor these tests into a test class and introduce a 
**class decorator**. The class decorator will automatically apply the profiling behavior to
the relevant test methods, eliminating the need to decorate each method individually.

This progression allows us to start with a simple function-level solution and then extend
the same concept to class-level profiling as the number of test methods grows.

```python
def profile(func=None, *, threshold=1, elapsed_time=True, stats=False, stats_limit=10):
    """Profile a function and optionally report its execution characteristics.
    Wraps a function with the :class:`Profiler` context manager to measure
    wall-clock execution time and, optionally, display the function-call
    statistics collected by ``cProfile``.

    The decorator can be used either directly as ``@profile`` or with
    configuration arguments such as ``threshold``, ``elapsed_time``, and
    ``stats``.

    Args:
        func: Function to be profiled. When ``None``, the decorator is being
            configured with keyword arguments and returns a partially
            configured decorator.
        threshold: Maximum expected execution time in seconds. A warning is
            displayed when the measured execution time exceeds this value.
        elapsed_time: Whether to display the wall-clock execution time after
            the function completes.
        stats: Whether to display the detailed function-call statistics
            collected by ``cProfile``.
        stats_limit: Maximum number of profiling entries to display when
            ``stats`` is enabled.

    Returns:
        A wrapped function that executes the original function under the
        configured profiling context.

    Example:
        Use the decorator with its default configuration::

            @profile
            def test_delayed_users(client):
                ...

        Configure the decorator explicitly::

            @profile(threshold=2, stats=True)
            def test_delayed_users(client):
                ...
    """
    if func is None:
        return partial(profile, threshold=threshold, elapsed_time=elapsed_time, stats=stats, stats_limit=stats_limit)

    @wraps(func)
    def wrapper(*args, **kwargs):
        """Execute the wrapped function within a profiling context.
        Starts the configured profiler before invoking the wrapped function and
        stops it after the function completes. Depending on the decorator
        configuration, the wrapper reports the elapsed execution time, checks
        the configured execution-time threshold, and displays the collected
        profiling statistics.

        Args:
            *args: Positional arguments passed to the wrapped function.
            **kwargs: Keyword arguments passed to the wrapped function.

        Returns:
            The value returned by the wrapped function.
        """
        with Profiler(limit=stats_limit) as p:
            result = func(*args, **kwargs)

        if elapsed_time:
            _elapsed_time = p.elapsed_time
            print(f"Time Elapsed {func.__name__}:{_elapsed_time:.3f} seconds")
            if _elapsed_time > threshold:
                print(f"WARNING: {func.__name__} took more than threshold limit of {threshold} seconds")

        if stats:
            p.print_stats()
        
        return result

    return wrapper
```


[Articles](../articles.md) \|  [Previous](../logging/logging.md)