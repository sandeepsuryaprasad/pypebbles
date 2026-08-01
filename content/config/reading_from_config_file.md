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

[TEST]
username = test_user
url = http://www.test.demo.com
port = 1234
host = testhost

[STAGE]
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
        return self._data.get(self._attr, "")

    def __getattr__(self, name):
        self._attr = name
        return self
    
    def __repr__(self):
        return f'Config("{self.section}")'
```
Suppose if we want to read the contents of the section `APPLE` here is what we 
can do,
```python
>>> config = Config("APPLE")
>>> config.year.value
'2026'
>>> config.model.value
'iPhone-17'
>>> config.manufacturer.value
'Apple Inc.'
>>> config.os_type.value
'iOS'
```
Similarly if we wanted to read a different section, create a separate instance
of `Config` object,

```python
>>> config = Config("GOOGLE")
>>> config.serial_number.value
'GGL987654321'
>>> config.os_version.value
'17'
>>> config.country.value
'US'
>>> config.device_name.value
'Demo Pixel.'
```
when you try to access the `section` data that does not exist, for example
```python
>>> config.phone_number.value  # this returns empty string
```
The magic happens in `__getattr__` method. when we try to access an attribute
on `config` object by saying `config.device_name.value`, the evaluation takes
place from left to right, so first python tries to look for an attribute
`device_name` on `config` object, since the attribute `device_name` does not
exist on `config` object, python calls `__getattr__` method by passing the name
of the attribute that we are trying to look up for in string format.

In the above case the name of the attribute that we are trying to access is
`device_name`. This attribute is passed as a string to `__getattr__` method.
`__getattr__` method is called only for a missing attribute. In `__getattr__`
method, I am setting instance attribute `_attr` that the user is trying to access
and it returns `self`.

So when `config.device_name` gets evaluated, `__getattr__` method sets the
name of the attribute that i am trying to look up for and returns the config object itself.

Now when a property `value` is called on `config.device_name`, the `value` property
does a dictionary look-up and returns the value of the attribute `device_name`.
The property `value` now knows the name of the attribute that it should pull out from
the dictionary, because `__getattr__` method would have already set the name
of the attribute in instance variable `_attr` when `config.device_name` was evaluated.

When we try to access the section data that does not exist by saying
`config.phone_number.value`, the name of the attribute is `phone_number` and
the `value` property tries to look-up for key `phone_number` in the dictionary
`self._data` which it cannot find and finally the `get` method returns an empty string.