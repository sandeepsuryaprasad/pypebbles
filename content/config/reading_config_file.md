## Turning Configuration Files into Python Objects 
**Last Updated:** August 1, 2026

Configuration files are commonly used to store application settings such as database details, 
API endpoints, credentials, feature flags, and environment-specific parameters.
Python provides built-in support for reading configuration files through modules 
such as `configparser`, making it straightforward to load and access these values.

In this article, we will learn how to read a `config.ini` file using Python's built-in 
`configparser` module and explore a clean, object-oriented approach to accessing 
configuration values.

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
The above config file has two different sections `APPLE` and `GOOGLE`, 
lets see how we can use an object oriented design approach in reading the 
config values in different sections of the file.

`config.py`

```python
from configparser import ConfigParser
from typing import Optional


class Config:
    """Provides a convenient way access to configuration values by section.
    The class reads configuration data from `config.ini` and exposes
    configuration keys through attribute access. A configuration section
    is selected when the instance is created, and individual values can
    then be accessed using the `value` property.

    Args:
        section: Name of the configuration section to access.

    Raises:
        KeyError: If the specified section does not exist in the
            configuration file.
    """
    def __init__(self, section):
        self.section: str = section.upper()
        self._parser: ConfigParser = self.parser
        self._data: dict = self.parse_to_dict
        self._attr: Optional[str] = None

    @property
    def parser(self):
        """Return a configured `ConfigParser` instance.
        The parser reads configuration data from ``config.ini``.
        """
        parser = ConfigParser()
        parser.read("config.ini")
        return parser

    @property
    def parse_to_dict(self):
        """Return the selected configuration section as a dictionary.
        Each configuration key in the selected section is stored as a
        key-value pair in the returned dictionary.

        Returns:
            A dictionary containing the configuration values for the
            selected section.

        Raises:
            KeyError: If the selected section does not exist.
        """
        if self.section not in self._parser.sections():
            raise KeyError(f"Invalid section {self.section}")
        config_data = {}
        for key, value in self._parser[self.section].items():
            config_data[key] = value
        return config_data

    @property
    def value(self):
        """Return the value of the most recently accessed configuration key.
        Returns:
            The configuration value associated with the accessed key,
            or an empty string if the key does not exist.
        """
        return self._data.get(self._attr, "")

    def __getattr__(self, name):
        """Capture access to an undefined attribute.
        The attribute name is stored internally and the current
        `Config` instance is returned, allowing configuration values
        to be accessed through expressions such as `config.device_name.value`.

        Args:
            name: Name of the configuration attribute being accessed.

        Returns:
            The current `Config` instance.
        """
        self._attr = name
        return self
    
    def __repr__(self):
        """Return the string representation of the configuration object.
        Returns:
            A string containing the selected configuration section.
        """
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
'Apple Inc'
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
'Demo Pixel'
```
when you try to access the `section` data that does not exist, for example
```python
>>> config.phone_number.value  # this returns empty string
```
The key mechanism is implemented through the `__getattr__` method. 
When an expression such as `config.device_name.value` is evaluated, Python resolves 
the attribute chain from left to right. It first attempts to resolve `device_name` 
on the `config` object. Because `device_name` is not explicitly defined as an 
attribute on the object, Python invokes `__getattr__`, passing the requested 
attribute name (`"device_name"`) as a string. This allows the implementation to 
dynamically resolve the attribute and continue the evaluation of the attribute chain.

Within the `__getattr__` method, we store the requested attribute name in the instance 
variable `_attr` and return the current `Config` instance (`self`). Returning `self`
allows the attribute lookup chain to continue, enabling expressions such as 
`config.device_name.value` to be evaluated dynamically.

When the `value` property is accessed on `config.device_name`, it performs a 
dictionary lookup using the attribute name stored in the instance variable `_attr` 
and returns the corresponding configuration value. The `_attr` variable is already 
populated during the evaluation of `config.device_name`, because Python invokes 
`__getattr__` when `device_name` cannot be resolved as a conventional attribute. 
This allows the `value` property to dynamically determine which key to retrieve 
from the underlying configuration dictionary.

When an attribute corresponding to a non-existent configuration key is accessed, 
such as `config.phone_number.value`, the `__getattr__` method stores `"phone_number"` 
in the instance variable `_attr`. The `value` property then attempts to retrieve the 
corresponding key from the underlying `self._data` dictionary. Since `"phone_number"`
is not present, the dictionary's `get()` method returns the specified default 
value—an empty string (`""`)—instead of raising a `KeyError`.

Finally, we implement the `__repr__` method to provide a clear and meaningful string 
representation of the `Config` object, making the object's identity and associated 
configuration section immediately apparent during debugging and interactive use.

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

### Taking inputs from terminal
We can further enhance our script by accepting the configuration section name 
as a command-line argument, allowing users to dynamically specify the section to 
load at runtime rather than hard-coding it in the application.

`main.py`

```python
import argparse
from config import Config

def console_entry():
    parser = argparse.ArgumentParser()
    parser.add_argument(
        "--section",
        dest="section",
        default="APPLE",
        help= "Name of the section in config.ini"
    )
    args = parser.parse_args()
    config = Config(args.section)
    return config


if __name__ == "__main__":
    config = console_entry()
    print(f"Serial No: {config.serial_number.value}")
    print(f"Model: {config.model.value}")
    print(f"Device Name: {config.device_name.value}")
```

Now we can pass the section name from terminal and will result in the following output
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
Device Name: Demo Pixel
```

In this article, we transformed a traditional configuration file into a clean, 
object-oriented interface using Python. Instead of exposing the underlying
`ConfigParser` implementation and requiring callers to work directly with sections 
and dictionary keys, we introduced a layer of abstraction that allows configuration 
values to be accessed through a more expressive object-oriented syntax.

A key part of this design was Python's `__getattr__` method. 
By intercepting attribute lookups for attributes that are not explicitly defined,
we were able to dynamically map attribute access to keys in the underlying 
configuration dictionary.

We also saw how returning `self` from `__getattr__` enables chained attribute 
access and how the `value` property ultimately resolves the requested configuration 
key from the underlying data.


Back to  [Articles](../articles.md)