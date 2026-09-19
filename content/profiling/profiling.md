[Articles](../articles.md) \|  [Previous](../logging/logging.md)

## Profiling in Python

In this article, we will build a reusable Python performance profiler that combines 
`cProfile` for function-level execution analysis with `perf_counter` for measuring 
wall-clock elapsed time and `process_time` for measuring the amount of CPU time consumed by 
the current process. The profiler will initially be implemented as a context manager 
and then extended using function decorators and class decorators, allowing the same 
profiling functionality to be applied to individual functions or entire classes with 
minimal changes to application code. 

The examples use API calls to demonstrate profiling in a realistic scenario, 
where the majority of the elapsed time may be spent waiting for a remote server 
rather than executing Python code locally. This distinction is important when analyzing
application performance because CPU execution time and end-to-end response time represent
different aspects of system performance.

The objective is not to simply measure how fast a piece of code runs, but to build a 
reusable profiling utility that can help identify performance characteristics and 
potential bottlenecks in real-world Python applications.

### Building a Reusable Profiler

Instead of placing `perf_counter`, `process_time` and `cProfile` calls directly inside every 
function that needs to be measured, we can encapsulate the profiling logic in a reusable class.

The Profiler class acts as a context manager, allowing profiling to be enabled 
automatically when entering a with block and disabled when leaving it. 
This keeps the profiling logic separate from the application code being measured.

The class combines two complementary approaches:

* `perf_counter` measures the total wall-clock time taken by an operation.
* `process_time` measures the amount of CPU time consumed by the current process. 
It **excludes** time during which the process is idle or waiting, such as waiting for I/O operations.
* `cProfile` collects detailed information about function calls made during that operation.
* `pstats.Stats` provides an interface for sorting, displaying, and exporting the 
collected profiling statistics.

This design allows the same profiler to be reused for API calls, file operations, 
database operations, or any other Python code where execution performance needs to 
be investigated.

The following implementation provides the basic profiling functionality. 
Each method has a specific responsibility: starting the profiler, stopping it, calculating 
elapsed time and process time, displaying profiling statistics, and exporting the collected 
statistics for further analysis.

```python
from dataclasses import dataclass
from pstats import Stats
from cProfile import Profile
from pstats import SortKey
from time import perf_counter, process_time


@dataclass
class ProfilerConfig:
    """Configuration settings for the :class:`Profiler`.

    Attributes:
        limit: Maximum number of profiling entries to display. Defaults to 10.
        sort_key: Sorting criterion for the profiling statistics.
            Defaults to :attr:`pstats.SortKey.CUMULATIVE`.
    """
    limit: int = 10
    sort_key: SortKey = SortKey.CUMULATIVE


class Profiler:
    """Profile code execution and collect performance statistics.
    Provides wall-clock time, CPU time, and ``cProfile`` function-call statistics
    through a context-manager interface. Profiling results can be displayed or
    exported for further analysis.

    Args:
        config: Optional :class:`ProfilerConfig` containing profiling settings.
        enable_profile: Optional :bool: Enable detailed cProfile statistics.
    """
    def __init__(self, *, config=None, enable_profile=False):
        self.config = config if config is not None else ProfilerConfig()
        self._start: float = 0.0
        self._end: float = 0.0
        self._cpu_start: float = 0.0
        self._cpu_end: float = 0.0
        self._profile: Profile = None
        self._stats: Stats = None
        self.enable_profile = enable_profile

    def __enter__(self):
        """Start timing and optionally enable detailed profiling."""
        self._start = perf_counter()
        self._cpu_start = process_time()
        if self.enable_profile:
            self._profile = Profile(builtins=False)
            self._profile.enable()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        """Stop timing and collect profiling statistics."""
        self._end = perf_counter()
        self._cpu_end = process_time()
        if self._profile:
            self._profile.disable()
            self._stats = Stats(self._profile)

    def print_profile_stats(self):
        """Display formatted function-call profiling statistics."""
        self._stats.strip_dirs().sort_stats(self.config.sort_key)
        self._stats.print_stats(self.config.limit)

    def print_execution_stats(self):
        """Display wall-clock and CPU execution times in seconds."""
        print()
        print("-" * 30)
        print(f"{'Time Elapsed':<13}: {self.elapsed_time:.3f} seconds")
        print(f"{'CPU Time':<13}: {self.cpu_time:.3f} seconds")
        print("-" * 30)

    @property
    def elapsed_time(self):
        """Return the elapsed wall-clock time in seconds, rounded to three decimals.
        Returns:
        float: Elapsed wall-clock time in seconds.
        """
        return round(self._end - self._start, 3)

    @property
    def cpu_time(self):
        """Return the CPU execution time in seconds, rounded to three decimals.
        Returns:
            float: CPU execution time consumed by the process in seconds.
        """
        return round(self._cpu_end - self._cpu_start, 3)

    def dump_stats(self, filename):
        """Save profiling statistics to a file.
        Args:
            filename: Path to the file where profiling statistics are saved.
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

For each API request, we can measure the total wall-clock elapsed time, CPU process time and, 
when required, inspect the function-call statistics collected by cProfile.

Let's start by applying the profiler explicitly to each API test. 
Let's Consider two test methods, `test_delayed_users` and `test_loop`, the first method 
invokes an HTTP `GET` to request against a delayed API endpoint and the second test method 
that calculates the sum of 100 million integers. We will use the `Profiler` class to measure 
the request's wall-clock execution time, CPU process time  and analyze the underlying function-call 
activity captured by `cProfile`.

```python
from os import environ
from httpx import Client
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


def test_loop():
    total = sum(i for i in range(0, 100000000))
    assert total == 4999999950000000
```
Before introducing any profiling , we will execute the `test_delayed_users` and `test_loop`
tests independently using `pytest`
```commandline
~$ pytest -vs profiler.py::test_delayed_users
====================================== test session starts ==============================
platform darwin -- Python 3.9.6, pytest-7.4.4, pluggy-1.3.0 -- /Library/Developer/CommandLineTools/usr/bin/python3
cachedir: .pytest_cache
rootdir: /Users/sandeepsuryaprasad/Documents/articles/profiler
plugins: anyio-4.12.1, instafail-0.5.0, trio-0.8.0, mock-3.12.0
collected 1 item

profiler.py::test_delayed_users PASSED
====================================== 1 passed in 2.33 =============================== 
```
```commandline
~$ pytest -vs profiler.py::test_loop         
====================================== test session starts ==============================
platform darwin -- Python 3.9.6, pytest-7.4.4, pluggy-1.3.0 -- /Library/Developer/CommandLineTools/usr/bin/python3
cachedir: .pytest_cache
rootdir: /Users/sandeepsuryaprasad/Documents/pro_tips/profiler
plugins: anyio-4.12.1, instafail-0.5.0, trio-0.8.0, mock-3.12.0
collected 1 item                                                                                                                                                                                        

profiler.py::test_loop PASSED
===================================== 1 passed in 2.52s =================================
```
The tests are passed. Now let's start by profiling the `test_delayed_users` and `test_loop` tests
by wrapping them inside the context manager.

```python
def test_delayed_users(client):
    with Profiler() as p:
        response = client.get("https://reqres.in/api/users?delay=2", headers=headers)
        assert response.status_code == 200
    p.print_execution_stats()


def test_loop():
    with Profiler() as p:
        total = sum(i for i in range(0, 100000000))
        assert total == 4999999950000000
    p.print_execution_stats()
```
```commandline
~$ pytest -vs profiler.py::test_delayed_users
====================================== test session starts ==============================
platform darwin -- Python 3.9.6, pytest-7.4.4, pluggy-1.3.0 -- /Library/Developer/CommandLineTools/usr/bin/python3
cachedir: .pytest_cache
rootdir: /Users/sandeepsuryaprasad/Documents/pro_tips/profiler
plugins: anyio-4.12.1, instafail-0.5.0, trio-0.8.0, mock-3.12.0
collected 1 item                                                                                                                                                                                        

profiler.py::test_delayed_users 
------------------------------
Time Elapsed : 2.208 seconds
CPU Time     : 0.008 seconds
------------------------------
PASSED
====================================== 1 passed in 2.32s =================================
```
```commandline
~$ pytest -vs profiler.py::test_loop         
====================================== test session starts ==============================
platform darwin -- Python 3.9.6, pytest-7.4.4, pluggy-1.3.0 -- /Library/Developer/CommandLineTools/usr/bin/python3
cachedir: .pytest_cache
rootdir: /Users/sandeepsuryaprasad/Documents/pro_tips/profiler
plugins: anyio-4.12.1, instafail-0.5.0, trio-0.8.0, mock-3.12.0
collected 1 item                                                                                                                                                                                        

profiler.py::test_loop 
------------------------------
Time Elapsed : 2.428 seconds
CPU Time     : 2.427 seconds
------------------------------
PASSED
====================================== 1 passed in 5.51s ==================================
```
This time the Elapsed Time and CPU Time is printed in the console for both the tests.

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
from httpx import Client
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

def test_loop():
    total = sum(i for i in range(0, 100000000))
    assert total == 4999999950000000
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
from functools import partial, wraps

def profile(func=None, *, enable_profile=False, execution_stats=True):
    """Profile a function and optionally display performance statistics.
    Args:
        func: Function to profile.
        enable_profile: Enable detailed cProfile profiling.
        execution_stats: Display execution timing statistics.
    Returns:
        The wrapped function.
    """
    if func is None:
        return partial(profile, enable_profile=enable_profile, execution_stats=execution_stats)

    @wraps(func)
    def wrapper(*args, **kwargs):
        with Profiler(enable_profile=enable_profile) as p:
            result = func(*args, **kwargs)
        if execution_stats:
            p.print_execution_stats()
        if enable_profile:
            p.print_profile_stats()
        return result
    return wrapper
```
### Applying the Profile Decorator
Now that we have implemented the profile decorator, let's apply it to the existing API tests.
```python
@profile
def test_resources(client):
    response = client.get("https://reqres.in/api/users?page=2", headers=headers)
    assert response.status_code == 200


@profile
def test_loop():
    total = sum(i for i in range(0, 100000000))
```
Let's run the above test using pytest.
```commandline
~$ pytest -vs profiler.py::test_delayed_users
============================================= test session starts =========================
platform darwin -- Python 3.9.6, pytest-7.4.4, pluggy-1.3.0 -- /Library/Developer/CommandLineTools/usr/bin/python3
cachedir: .pytest_cache
rootdir: /Users/sandeepsuryaprasad/Documents/pro_tips/profiler
plugins: anyio-4.12.1, instafail-0.5.0, trio-0.8.0, mock-3.12.0
collected 1 item                                                                                                             

profiler.py::test_delayed_users
------------------------------
Time Elapsed : 2.435 seconds
CPU Time     : 0.015 seconds
------------------------------
PASSED
============================================ 1 passed in 2.56s ============================ 
```
```commandline
~$ pytest -vs profiler.py::test_loop         
============================================ test session starts ==========================
platform darwin -- Python 3.9.6, pytest-7.4.4, pluggy-1.3.0 -- /Library/Developer/CommandLineTools/usr/bin/python3
cachedir: .pytest_cache
rootdir: /Users/sandeepsuryaprasad/Documents/pro_tips/profiler
plugins: anyio-4.12.1, instafail-0.5.0, trio-0.8.0, mock-3.12.0
collected 1 item                                                                                                             

profiler.py::test_loop 
------------------------------
Time Elapsed : 2.446 seconds
CPU Time     : 2.437 seconds
------------------------------
PASSED
========================================= 1 passed in 2.53s ==============================
```
Let's print the profile stats 
```python
@profile(enable_profile=True)
def test_delayed_users(client):
    with Profiler(enable_profile=True) as p:
        response = client.get("https://reqres.in/api/users?delay=2", headers=headers)
        assert response.status_code == 200
```
```commandline
~$ pytest -vs profiler.py::test_delayed_users
============================================= test session starts ========================
platform darwin -- Python 3.9.6, pytest-7.4.4, pluggy-1.3.0 -- /Library/Developer/CommandLineTools/usr/bin/python3
cachedir: .pytest_cache
rootdir: /Users/sandeepsuryaprasad/Documents/pro_tips/profiler
plugins: anyio-4.12.1, instafail-0.5.0, trio-0.8.0, mock-3.12.0
collected 1 item                                                                                      

profiler.py::test_delayed_users 
------------------------------
Time Elapsed : 2.443 seconds
CPU Time     : 0.013 seconds
------------------------------
         4 function calls in 2.443 seconds

   Ordered by: cumulative time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.000    0.000    2.443    2.443 profiler.py:165(test_delayed_users)
        1    2.443    2.443    2.443    2.443 profiler.py:64(__enter__)
        1    0.000    0.000    0.000    0.000 profiler.py:54(__init__)
        1    0.000    0.000    0.000    0.000 <string>:2(__init__)

PASSED
============================================= 1 passed in 2.56s ============================
```
```commandline
~$ pytest -vs profiler.py::test_loop         
============================================= test session starts ===========================
platform darwin -- Python 3.9.6, pytest-7.4.4, pluggy-1.3.0 -- /Library/Developer/CommandLineTools/usr/bin/python3
cachedir: .pytest_cache
rootdir: /Users/sandeepsuryaprasad/Documents/pro_tips/profiler
plugins: anyio-4.12.1, instafail-0.5.0, trio-0.8.0, mock-3.12.0
collected 1 item                                                                                      

profiler.py::test_loop 
------------------------------
Time Elapsed : 6.492 seconds
CPU Time     : 6.482 seconds
------------------------------
         100000003 function calls in 6.492 seconds

   Ordered by: cumulative time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    3.501    3.501    6.492    6.492 profiler.py:172(test_loop)
100000001    2.991    0.000    2.991    0.000 profiler.py:174(<genexpr>)
        1    0.000    0.000    0.000    0.000 profiler.py:72(__exit__)

PASSED
============================================ 1 passed in 6.58s ================================
```
**Observation:** When the tests are executed without profiling, execution time is less than 
when the same tests are executed under cProfile. The increase is caused by the 
**overhead introduced by collecting detailed profiling information**. 

So enable profiling only if it is needed. If you are interested only in measuring 
total wall-clock time and CPU time, do not turn on the profiling switch by enabling `enable_profile`.

Consider a test class containing several test methods. 
```python
class TestUsers:
    def test_single_user(self, client):
        response = client.get("https://reqres.in/api/users/2", headers=headers)
        assert response.status_code == 200

    def test_user_not_found(self, client):
        response = client.get("https://reqres.in/api/users/23", headers=headers)
        assert response.status_code == 404

    def test_list_users(self, client):
        response = client.get("https://reqres.in/api/users?page=2", headers=headers)
        assert response.status_code == 200

    def test_resources(self, client):
        response = client.get("https://reqres.in/api/users?page=2", headers=headers)
        assert response.status_code == 200

    def test_delayed_users(self, client):
        response = client.get("https://reqres.in/api/users?delay=2", headers=headers)
        assert response.status_code == 200

    def test_more_delayed_users(self, client):
        response = client.get("https://reqres.in/api/users?delay=3", headers=headers)
        assert response.status_code == 200
```
If we want to profile all of those methods, we would have to apply the `@profile` 
decorator to each method individually. This introduces unnecessary repetition and 
requires us to remember to decorate every new test method added to the class.

A better approach is to apply the profiling behavior at the class level. 
A class decorator can inspect the methods defined in a class and automatically apply 
the profile decorator to the methods that need to be profiled.

Let's implement a `profile_class` decorator that automatically profiles the test methods 
in that class.

### Introducing the Class Decorator

```python
from functools import partial


def profile_class(cls=None, *, execution_stats=True, profile_stats=False):
    """Profile callable methods in a class.
    Applies the :func:`profile` decorator to each callable method defined
    directly in the class.

    Args:
        cls: Class whose methods are to be profiled.
        execution_stats: Display execution timing statistics.
        profile_stats: Display detailed profiling statistics.

    Returns:
        The decorated class.
    """
    if cls is None:
        return partial(profile_class, execution_stats=execution_stats, profile_stats=profile_stats)

    def _decorate_each_method(method):
        """Apply the profiling decorator to a class method."""
        return profile(method, execution_stats=execution_stats, profile_stats=profile_stats)

    for name, value in cls.__dict__.items():
        if callable(value) and not name.startswith("__"):
            setattr(cls, name, _decorate_each_method(value))

    return cls
```
Now let's apply the above class decorator the our test class `TestUsers`

```python
@profile_class
class TestUsers:
    def test_single_user(self, client):
        response = client.get("https://reqres.in/api/users/2", headers=headers)
        assert response.status_code == 200

    def test_user_not_found(self, client):
        response = client.get("https://reqres.in/api/users/23", headers=headers)
        assert response.status_code == 404

    def test_list_users(self, client):
        response = client.get("https://reqres.in/api/users?page=2", headers=headers)
        assert response.status_code == 200

    def test_resources(self, client):
        response = client.get("https://reqres.in/api/users?page=2", headers=headers)
        assert response.status_code == 200

    def test_delayed_users(self, client):
        response = client.get("https://reqres.in/api/users?delay=2", headers=headers)
        assert response.status_code == 200

    def test_more_delayed_users(self, client):
        response = client.get("https://reqres.in/api/users?delay=3", headers=headers)
        assert response.status_code == 200
```
When we execute the above test class, the class-level decorator automatically profiles 
each test method and produces the following output.
```commandline
~$ pytest -vs profiler.py::TestUsers
===================================== test session starts ============================================
platform darwin -- Python 3.9.6, pytest-7.4.4, pluggy-1.3.0 -- /Library/Developer/CommandLineTools/usr/bin/python3
cachedir: .pytest_cache
rootdir: /Users/sandeepsuryaprasad/Documents/pro_tips/profiler
plugins: anyio-4.12.1, instafail-0.5.0, trio-0.8.0, mock-3.12.0
collected 6 items                                                                                                                                                                                       

profiler.py::TestUsers::test_single_user Time Elapsed test_single_user:0.447 seconds
         2498 function calls (2446 primitive calls) in 0.447 seconds

   Ordered by: cumulative time
   List reduced from 459 to 2 due to restriction <2>

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.000    0.000    0.447    0.447 profiler.py:209(test_single_user)
        1    0.000    0.000    0.447    0.447 _client.py:1036(get)

PASSED
profiler.py::TestUsers::test_user_not_found Time Elapsed test_user_not_found:0.522 seconds
         1621 function calls in 0.522 seconds

   Ordered by: cumulative time
   List reduced from 279 to 2 due to restriction <2>

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.000    0.000    0.522    0.522 profiler.py:213(test_user_not_found)
        1    0.000    0.000    0.522    0.522 _client.py:1036(get)

PASSED
profiler.py::TestUsers::test_list_users Time Elapsed test_list_users:0.505 seconds
         1710 function calls (1709 primitive calls) in 0.505 seconds

   Ordered by: cumulative time
   List reduced from 281 to 2 due to restriction <2>

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.000    0.000    0.505    0.505 profiler.py:217(test_list_users)
        1    0.000    0.000    0.505    0.505 _client.py:1036(get)

PASSED
profiler.py::TestUsers::test_resources Time Elapsed test_resources:0.057 seconds
         1727 function calls (1726 primitive calls) in 0.057 seconds

   Ordered by: cumulative time
   List reduced from 279 to 2 due to restriction <2>

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.000    0.000    0.057    0.057 profiler.py:221(test_resources)
        1    0.000    0.000    0.057    0.057 _client.py:1036(get)

PASSED
profiler.py::TestUsers::test_delayed_users Time Elapsed test_delayed_users:2.246 seconds
         1711 function calls (1710 primitive calls) in 2.246 seconds

   Ordered by: cumulative time
   List reduced from 281 to 2 due to restriction <2>

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.000    0.000    2.246    2.246 profiler.py:225(test_delayed_users)
        1    0.000    0.000    2.246    2.246 _client.py:1036(get)

PASSED
profiler.py::TestUsers::test_more_delayed_users Time Elapsed test_more_delayed_users:3.322 seconds
WARNING: test_more_delayed_users took more than threshold limit of 2.5 seconds
         1711 function calls (1710 primitive calls) in 3.322 seconds

   Ordered by: cumulative time
   List reduced from 281 to 2 due to restriction <2>

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.000    0.000    3.322    3.322 profiler.py:229(test_more_delayed_users)
        1    0.000    0.000    3.322    3.322 _client.py:1036(get)

PASSED
======================================= 6 passed in 7.28s ========================================
```
### Limitation of class decorator
Although the class decorator significantly reduces repetitive code by applying the 
profiling configuration to multiple test methods, it introduces a 
limitation, **the same profiling configuration is applied to all decorated methods 
in the class**.

For example, if the class decorator is configured with `threshold=5`, every test 
method decorated by the class decorator will use the same five-second threshold. 
We cannot directly specify a different threshold for individual test methods through 
the class decorator.

A test suite may contain operations with significantly different expected execution times. 
For example, a test that performs a simple API request might reasonably have a threshold 
of two seconds, while another test involving a large file upload or download might 
require a higher threshold.

In such cases, the function decorator offers greater flexibility because each method 
can be configured independently.

```python
@profile(threshold=2)
def test_single_user(self):
    ...

@profile(threshold=10)
def test_large_file_download(self):
    ...
```

Therefore, the two approaches serve different purposes:

* **Function decorator** — provides fine-grained, method-level configuration.
* **Class decorator** — provides convenient, consistent configuration across multiple methods.

The choice between them depends on whether the primary requirement is **individual control** or **convenient class-level configuration**.

### Final Thoughts
In this article, we started by building a reusable Profiler class that combines 
`time.perf_counter` and `time.process_time` for measuring wall-clock execution time and 
cpu time with `cProfile` for collecting function-call statistics. 
We then used the profiler as a context manager to explicitly define the portion of code
that should be measured.

As the number of tests increased, we saw that adding profiling logic directly to 
every test introduced unnecessary repetition. We addressed this by implementing a 
function decorator that allowed profiling to be added to existing functions without
embedding profiling logic into their implementation.

Finally, we extended the same approach to a class decorator. 
Instead of decorating every test method individually, the class decorator can 
automatically apply the profiling behavior to the relevant methods in the class. 
This provides a more scalable solution when profiling a larger collection of tests.

[Articles](../articles.md) \|  [Previous](../logging/logging.md)