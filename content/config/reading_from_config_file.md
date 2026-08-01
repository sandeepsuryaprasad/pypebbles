## Object oriented approach for reading `config.ini` file

In this section we will learn how to read a configuration file, let's say `config.ini` using python's built-in
`configparser` module

Lets consider a `config.ini` with the following sections and contents.
```commandline
[APPLE]
serial_number = APL123456789
model = iPhone-17
device_name = Demo iPhone
os_type = iOS
os_version = 16
year = 2026
manufacturer = Apple Inc.
country = US

[GOOGLE]
serial_number = GGL987654321
model = Pixel-10
device_name = Demo Pixel.
os_type = Android
os_version = 17
year = 2026
manufacturer = Google LLC
country = US

[TEST.SETTINGS]
username = test_user
url = http://www.test.demo.com
port = 1234
host = testhost

[STAGE.SETTINGS]
username = stage_user
url = http://www.stage.demo.com
port = 8080
host = stagehost

[API]
timeout = 30
baseurl = http://www.baseurl.com

[DATABASE]
port = 9090
```
The above config file has 6 different sections, lets see how we can use an
object oriented design approach in reading the config values in different
sections of the file.
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