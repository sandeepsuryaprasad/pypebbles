[Articles](../articles.md) \|  [Previous](../json/reading_json.md) \| [Next](../profiling/profiling.md)

## Building a Flexible Logging Abstraction
**Last Updated:** August 8, 2026

In this article, we will build a lightweight logging abstraction around Python's 
built-in logging module. Rather than replacing the standard logging framework, 
our goal is to encapsulate its configuration behind a simple, reusable interface.

We will gradually build a Logger class that handles common logging concerns such as log 
levels, handlers, formatters, and logger configuration. 
We will also allow the logging level to be controlled through command-line arguments, 
making the logger more practical for real-world applications.

The objective of this article is not to recreate Python's logging framework. 
Instead, it is to demonstrate how a well-designed abstraction can simplify an existing API while 
keeping the underlying functionality intact.

`mylogger.py`
```python
import logging
from typing import Optional

class Logger:
    _VALID_LOG_LEVELS = {
        logging.DEBUG,
        logging.INFO,
        logging.WARNING,
        logging.ERROR,
        logging.CRITICAL,
    }
    _LOG_FORMAT = "[%(levelname)s] [%(asctime)s]  %(message)s"

    def __init__(
            self, name: str,
            level: int = logging.INFO,
            handler: Optional[logging.Handler] = None
    ):
        self.level = level
        self.handler = handler
        self.logger = self._set_logger(name)

    @property
    def level(self):
        """Return the logging level configured for the logger."""
        return self._level

    @level.setter
    def level(self, value):
        """
        Set the logging level after validating the supplied value.
        Raises:
            ValueError: If the supplied logging level is not supported.
        """
        if value not in self._VALID_LOG_LEVELS:
            raise ValueError(f"Invalid logging level {value}")
        self._level = value

    @property
    def handler(self):
        """Return the logging handler configured for the logger."""
        return self._handler

    @handler.setter
    def handler(self, value):
        """
        Set the logging handler.
        """
        self._handler = value if value else logging.StreamHandler()

    def _set_logger(self, name):
        """
        Create and configure a named logger.
        Sets the logger's level, creates a formatter using the configured
        log format, and applies the formatter to the configured handler.
        The handler is added only when the logger has no existing handlers.
        Args:
            name: Name used to retrieve the logger.
        Returns:
            A configured :class:`logging.Logger` instance.
        """
        logger = logging.getLogger(name)
        logger.setLevel(self._level)
        formatter = logging.Formatter(self._LOG_FORMAT)
        self._handler.setFormatter(formatter)
        if not logger.handlers:
            logger.addHandler(self._handler)
        return logger

    def __getattr__(self, name):
        """
        Delegate unknown attribute lookups to the underlying logger.

        This allows methods such as ``debug()``, ``info()``, ``warning()``,
        ``error()``, and ``critical()`` to be called directly on the wrapper
        without explicitly defining each method.

        Args:
            name: Name of the attribute being accessed.

        Returns:
            The corresponding attribute from the underlying logger.
        """
        return getattr(self.logger, name)
```

We delegate `info`, `debug`, `warning` and `critical` attributes to `self.logger` 
object, which is instance of original python's built-in `logger` 
object using `__getattr__` method.

Let's look at an example on how we can use the above class. For demonstration purpose consider
a python module `add.py` with simple function that adds two numbers, 

`add.py`
```python
def add(a: int, b: int):
    return a + b
```
We want to add log messages to the above function so that the messages will be printed in terminal.
Here is how we can do it,

```python
from mylogger import Logger

logger = Logger(__name__)   # by default log level will be set to `INFO`

def add(a: int, b: int):
    logger.info(f"add function called with args 'a': {a} and 'b': {b}")
    return a + b

print(add(1, 2))   # calling our add function
```
When we run `add.py` from terminal, we get the below output, which is correct.
```commandline
~$ python3 add.py
[2026-08-08 08:07:07,456] [INFO]  Calling add function with args 'a': 1 and 'b': 2
3
```
`INFO` message is emitted in the terminal along with the actual result of addition.

Now let's say someone is calling `add` function by passing arguments in a string format, 
something like below,

```python
from mylogger import Logger

logger = Logger(__name__)

def add(a: int, b: int):
    logger.info(f"Calling add function with args 'a': {a} and 'b': {b}")
    return a + b

print(add("1", "2"))   # calling our add function by passing 1 and 2 in string format
```
when we run the above program in terminal, we get the below output,
```commandline
~$ python3 add.py
[2026-08-09 08:22:08,788] [INFO]  Calling add function with args 'a': 1 and 'b': 2
12
```
That's really interesting thing to debug and see what's really happening. The problem
in the above function call is that instead of passing numbers to `add` function, we are passing
numbers in `str` format, So the `add` function instead of adding two numbers it is concatenating.

Now we want to know what's really happening in the code and why is it giving the output
that it is giving. For debugging purpose, we want our function to output some extra information.

Let's modify our `add` function to output more information in case if we ran our code in debug
mode.
```python
from logging_demo import Logger

logger = Logger(__name__)   # default log level is still `INFO`

def add(a: int, b: int):
    logger.info(f"Calling add function with args 'a': {a} and 'b': {b}")
    logger.debug("Type of arg 'a': {type(a)} and 'b': type(b)")
    return a + b

print(add("1", "2"))
```
In the above modified code, we have added a debug log which outputs the `type` of 
arguments `a` and `b`. By default, this debug log will not be emitted to terminal if 
we ran the code without setting the log level to `DEBUG`. If we run the code now, 
we still get the below output,
```commandline
~$ python3 add.py
[2026-08-08 09:28:37,721] [INFO]  Calling add function with args 'a': 1 and 'b': 2
12
```
To get the debug logs, we have to modify our `add.py` file and change the log level while 
creating instance of `Logger` class. Below is the modified code from `add.py` file,

```python
import logging
from logging_demo import Logger

logger = Logger(__name__, level=logging.DEBUG)  # changing the log level to `DEBUG`

def add(a: int, b: int):
    logger.info(f"Calling add function with args 'a': {a} and 'b': {b}")
    logger.debug("Type of arg 'a': {type(a)} and 'b': type(b)")
    return a + b

print(add("1", "2"))
```
Now when we run `add.py` from terminal, we get the below output,
```commandline
~$ python3 z_add.py
[INFO] [2026-08-08 09:39:37,464]  Calling add function with args 'a': 1 and 'b': 2
[DEBUG] [2026-08-08 09:39:37,464]  Type of arg 'a': <class 'str'> and 'b': <class 'str'>
12
```
From the above `DEBUG` message it is very clear that the type of argument `a` is `str` and 
type of argument `b` is `str` and that is why `add` function is concatenating `a` and `b` instead
of adding.

But the problem with the above mechanism is we need to modify the code to change the log level from
`INFO` to `DEBUG`. Once we rectify the problem, we need to revert log level back to `DEBUG` for which
again we need to modify the code. 

Our intention here is to emit only `INFO` messages to terminal during normal run and `DEBUG` 
messages when we want to run our code in debug mode. We do not want all messages to show up
in terminal without explicitly requesting for it. 

But from the above code it is very clear that if we have to switch between `DEBUG` and `INFO`
level, we have make changes in the code. 

The elegant solution for this problem is to control the log level from terminal.

### Setting log level from CLI
The logging level should be configurable through a command-line argument when 
executing the script. For example, specifying the `--debug` option should enable the 
`DEBUG` logging level, allowing debug messages to be emitted to the terminal. 
When the `--debug` option is not specified, the application should default to the 
`INFO` logging level and emit only `INFO` and higher-severity messages.

Below is the mechanism that we are looking for,
```commandline
~$ python3 add.py --debug
[2026-08-09 13:41:56,925] [INFO]  Calling add function with args 'a': 1 and 'b': 2
[2026-08-09 13:41:56,925] [DEBUG]  The type of arg 'a': <class 'str'> and 'b': <class 'str'>
12
```
When executing `add.py` from the terminal, we can optionally specify the `--debug`
command-line argument. If `--debug` is provided, the logging level should be configured 
as `DEBUG` within the `Logger` class defined in `mylogger.py`, allowing debug messages 
to be emitted. When `add.py` is executed without the `--debug` argument, the logger 
should default to the `INFO` level and emit only `INFO` and higher-severity messages.

```commandline
~$ python3 add.py        
[2026-08-09 13:47:07,640] [INFO]  Calling add function with args 'a': 1 and 'b': 2
12
```

To implement this mechanism, we will no longer configure the logging level through the 
`level` property setter. Instead, we will define a separate function outside the
`Logger` class to process the command-line arguments and determine the appropriate 
logging level. The resulting log level will then be used to initialize the `Logger` instance.

Here is solution,
```python
import logging
from typing import Optional
import argparse


def get_log_level_from_terminal():
    """Determine the logging level from command-line arguments.

    Parses the command-line arguments and returns `DEBUG` when the
    `--debug` option is specified. Otherwise, `INFO` is returned
    as the default logging level.

    Returns:
        The logging level selected from the command-line arguments.
    """
    parser = argparse.ArgumentParser(description="Set the logging level via command line")
    parser.add_argument('--debug', action="store_true",
                        help='Set the logging level (DEBUG, INFO)')
    args = parser.parse_args()
    
    return logging.DEBUG if args.debug else logging.INFO


class Logger:
    _VALID_LOG_LEVELS = {
        logging.DEBUG,
        logging.INFO,
        logging.WARNING,
        logging.ERROR,
        logging.CRITICAL,
    }
    _LOG_FORMAT = "[%(levelname)s] [%(asctime)s]  %(message)s"

    def __init__(
            self, name: str,
            handler: Optional[logging.Handler] = None
    ):
        self._level = get_log_level_from_terminal(), # setting log level from CLI input 
        self.handler = handler
        self.logger = self._set_logger(name)

    @property
    def level(self):
        """Return the logging level configured for the logger."""
        return self._level

    @property
    def handler(self):
        """Return the logging handler configured for the logger."""
        return self._handler

    @handler.setter
    def handler(self, value):
        """
        Set the logging handler.
        """
        self._handler = value if value else logging.StreamHandler()

    def _set_logger(self, name):
        """
        Create and configure a named logger.
        Sets the logger's level, creates a formatter using the configured
        log format, and applies the formatter to the configured handler.
        The handler is added only when the logger has no existing handlers.
        Args:
            name: Name used to retrieve the logger.
        Returns:
            A configured :class:`logging.Logger` instance.
        """
        logger = logging.getLogger(name)
        logger.setLevel(self._level)
        formatter = logging.Formatter(self._LOG_FORMAT)
        self._handler.setFormatter(formatter)
        if not logger.handlers:
            logger.addHandler(self._handler)
        return logger

    def __getattr__(self, name):
        """
        Delegate unknown attribute lookups to the underlying logger.

        This allows methods such as ``debug()``, ``info()``, ``warning()``,
        ``error()``, and ``critical()`` to be called directly on the wrapper
        without explicitly defining each method.

        Args:
            name: Name of the attribute being accessed.

        Returns:
            The corresponding attribute from the underlying logger.
        """
        return getattr(self.logger, name)
```

In the above code we have made few changes, 
* We are no longer configuring the logging level through a property setter. 
The `@level.setter` method has been removed from the `Logger` class, leaving 
only the getter to provide read-only access to the configured logging level.

* In the `__init__` method, the instance variable `self._level` is initialized 
by calling `get_log_level_from_terminal()`, which determines the logging level 
based on the command-line arguments supplied when the application is executed.


[Articles](../articles.md) \|  [Previous](../json/reading_json.md) \| [Next](../profiling/profiling.md)