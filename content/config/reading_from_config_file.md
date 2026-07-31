## Object oriented approach for reading `config.ini` file

In this section we will learn how to read a configuration file, let's say `config.ini` using python's built-in
`configparser` module

```python
from configparser import ConfigParser
from typing import Optional

class Config:
    def __init__(self, section):
        self.section: str = section.upper()
        self._parser: ConfigParser = self.parser
        self._data: dict = self.parse_to_dict
        self._attr: Optional[str] = None

    @property
    def parser(self):
        parser = ConfigParser()
        parser.read("config.ini")
        return parser

    @property
    def parse_to_dict(self):
        if self.section not in self._parser.sections():
            raise KeyError(f"Invalid section {self.section}")
        config_data = {}
        for key, value in self._parser[self.section].items():
            config_data[key] = value
        return config_data

    @property
    def value(self):
        return self._data.get(self._attr)

    def __getattr__(self, name):
        self._attr = name
        return self
    
    def __repr__(self):
        return f'Config("{self.section}")'
```
This is demo