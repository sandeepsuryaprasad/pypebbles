## Object-Oriented Approach for Logging

In this article we will implement objected-oriented approach for `logging`
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
            level: int = logging.DEBUG,
            handler: Optional[logging.Handler] = None
    ):
        self.level = level
        self.handler = handler
        self.logger = self._set_logger(name)

    @property
    def level(self):
        return self._level

    @level.setter
    def level(self, value):
        if value not in self._VALID_LOG_LEVELS:
            raise ValueError(f"Invalid logging level {value}")
        self._level = value

    @property
    def handler(self):
        return self._handler

    @handler.setter
    def handler(self, value):
        self._handler = value if value else logging.StreamHandler()

    def _set_logger(self, name):
        logger = logging.getLogger(name)
        logger.setLevel(self._level)
        formatter = logging.Formatter(self._LOG_FORMAT)
        self._handler.setFormatter(formatter)
        if not logger.handlers:
            logger.addHandler(self._handler)
        return logger

    def info(self, message):
        self.logger.info(message)

    def debug(self, message):
        self.logger.debug(message)

    def warning(self, message):
        self.logger.warning(message)

    def critical(self, message):
        self.logger.critical(message)
```

In the above class implementation, the functions `info`, `debug`, `warning` and `critical`
are just a wrapper on top of original `logger` object from python's built-in library.
Instead of having to write wrappers, we can delegate these attributes to `self.logger` object,
which is instance of original python's built-in `logger` object using `__getattr__` method.

### Delegating un-known attribute look-up using `__getattr__`

The final solution may look somthing like this,
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
            level: int = logging.DEBUG,
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
Back to  [Articles](../articles.md)