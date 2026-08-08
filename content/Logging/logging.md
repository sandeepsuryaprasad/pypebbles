### To be edited
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

Back to  [Articles](../articles.md)