## Reading Configuration File Using Object-Oriented Approach 
**Last Updated:** August 1, 2026

In this section we will learn how to read a configuration file, let's say `config.ini` using python's built-in
`configparser` module

Lets consider the following config file.

`config.ini`

```commandline
[APPLE]
serial_number = APL123456789
model = iPhone-17
device_name = Demo iPhone
os_type = iOS
os_version = 16
year = 2026
manufacturer = Apple Inc
country = US

[GOOGLE]
serial_number = GGL987654321
model = Pixel-10
device_name = Demo Pixel
os_type = Android
os_version = 17
year = 2026
manufacturer = Google LLC
country = US
```
The above config file has 2 different sections `APPLE` and `GOOGLE`, 
lets see how we can use an object oriented design approach in reading the 
config values in different sections of the file.

`config.py`

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
`__getattr__` method gets called only for a missing attributes. In `__getattr__`
method, we are setting instance attribute `_attr` that the user is trying to access
and finally we `self`.

So when `config.device_name` gets evaluated, `__getattr__` method sets the
name of the attribute that we are trying to look up for and returns the config object itself.

Now when a property `value` is called on `config.device_name`, the `value` property
does a dictionary look-up and returns the value of the attribute `device_name`.
The property `value` now knows the name of the attribute that it should pull out from
the dictionary, because `__getattr__` method would have already set the name
of the attribute in instance variable `_attr` when `config.device_name` was evaluated.

When we try to access the section data that does not exist by saying
`config.phone_number.value`, the name of the attribute is `phone_number` and
the `value` property tries to look-up for key `phone_number` in the dictionary
`self._data` which it cannot find and finally the `get` method returns an empty string.

And finally we have a nice string representation of our `config` object.
```python
>>> config = Config("APPLE")
>>> print(config)
Config("APPLE")
```
```python
>>> config = Config("GOOGLE")
>>> print(config)
Config("GOOGLE")
```

#### Taking inputs from terminal
We can further enhance our script take inputs from terminal.

`main.py`

```python
import argparse
from config import Config

def console_entry():
    _parser = argparse.ArgumentParser()
    _parser.add_argument(
        "--section",
        dest="section",
        default="APPLE",
        help= "Name of the section in config.ini"
    )
    parser = _parser.parse_args()
    config = Config(parser.section)
    return config


if __name__ == "__main__":
    config = console_entry()
    print(f"Serial No: {config.serial_number.value}")
    print(f"Model: {config.model.value}")
    print(f"Device Name: {config.device_name.value}")
```

Now we can pass the section name from terminal and will result in the 
following output

```commandline
~$ python3 main.py --section APPLE
Serial No: APL123456789
Model: iPhone-17
Device Name: Demo iPhone
```
```commandline
~$ python3 main.py --section GOOGLE
Serial No: GGL987654321
Model: Pixel-10
Device Name: Demo Pixel.
```


Back to  [Articles](../articles.md)