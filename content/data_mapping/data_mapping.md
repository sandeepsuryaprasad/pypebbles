## Transforming Mappings into Semantic Attribute Interfaces

Python dictionaries provide a convenient way to organize and retrieve 
related information using key-value pairs. They are commonly used to represent lookup tables,
configuration data, constants, and other structured information.

Consider an HTTP status-code mapping where numeric status codes are associated with their 
corresponding symbolic names. To retrieve a status code, we typically use dictionary-based 
lookups such as `http_status_codes[404]`. While this approach is straightforward, 
accessing values through dictionary keys may become less expressive when working with large
mappings or frequently referenced entries.

Python's dynamic attribute access capabilities provide an alternative. By exposing mapping 
entries as object attributes, we can access values using meaningful names and familiar dot
notation. For example, `http_codes.not_found` is more descriptive and intuitive than repeatedly
looking up values using dictionary keys.

In this article, we explore how to transform ordinary Python mappings into semantic attribute
interfaces using dynamic attribute creation. We will begin with HTTP status codes and then apply
the same concept to other practical examples, including international telephone calling
codes and currency information.

### HTTP Status Code Mapping

HTTP status codes indicate the outcome of an HTTP request. Each status code is represented 
by a three-digit number and belongs to one of five categories: informational responses, 
successful responses, redirections, client errors, and server errors.

In Python, we can represent these status codes using a dictionary, where each numeric code maps
to one or more descriptive names. These names act as aliases, allowing the same status code 
to be referenced using different symbolic identifiers.

Let's define a sample mapping covering commonly used HTTP status codes:
```python
http_status_codes = {
    # Informational
    100: ("continue",),
    101: ("switching_protocols",),
    
    # Success
    200: ("ok", "success"),
    201: ("created",),
    202: ("accepted",),
    204: ("no_content",),

    # Redirection
    301: ("moved_permanently",),
    302: ("found", "redirect"),
    304: ("not_modified",),

    # Client errors
    400: ("bad_request",),
    401: ("unauthorized",),
    403: ("forbidden",),
    404: ("not_found",),
    405: ("method_not_allowed",),
    408: ("request_timeout",),
    409: ("conflict",),
    429: ("too_many_requests",),

    # Server errors
    500: ("internal_server_error",),
    501: ("not_implemented",),
    502: ("bad_gateway",),
    503: ("service_unavailable",),
    504: ("gateway_timeout",),
}
```

### Building an Attribute-Based Lookup

To expose the HTTP status-code mapping through attribute access, we can create a small Python 
class called `HttpCodeLookup`

The class accepts the status-code dictionary during initialization and dynamically creates 
an attribute for every symbolic name associated with a status code.

```python
import keyword

class HttpCodeLookup:
    """Expose HTTP status codes through dynamically created attributes.

    This class transforms an HTTP status-code mapping into an object whose
    attributes represent the symbolic names associated with each status code.

    Each numeric status code can have one or more symbolic names. During
    initialization, an attribute is created for every name, with the
    corresponding numeric status code assigned as its value.

    For example, given the mapping::

        {
            200: ("ok", "success"),
            404: ("not_found",),
        }

    The resulting object supports attribute-based access::

        http.ok         # 200
        http.success    # 200
        http.not_found  # 404

    Python keywords used as mapping keys are normalized by appending an
    underscore to make them suitable for attribute access.

    Args:
        http_codes (dict): A mapping of HTTP status codes to tuples of
            their corresponding symbolic names.

    Example:
        >>> http = HttpCodeLookup({
        ...     200: ("ok", "success"),
        ...     404: ("not_found",),
        ... })
        >>> http.ok
        200
        >>> http.not_found
        404
    """

    def __init__(self, http_codes: dict):
        """Initialize the lookup object from an HTTP status-code mapping.

        Iterates through the supplied mapping and dynamically creates an
        instance attribute for each symbolic name. Each attribute is assigned
        the numeric HTTP status code associated with that name.

        Multiple symbolic names associated with the same status code are
        created as separate attributes.

        Args:
            http_codes (dict): A mapping where keys are numeric HTTP status
                codes and values are iterables containing their symbolic names.
        """
        for code, titles in http_codes.items():
            for title in titles:
                setattr(self, self._normalize(title), code)

    @staticmethod
    def _normalize(key: str) -> str:
        """Normalize a mapping key for Python attribute access.

        Checks whether the supplied key is a reserved Python keyword.
        If it is, an underscore is appended to the key to avoid conflicts
        when accessing the corresponding attribute using dot notation.

        Keys that are not Python keywords are returned unchanged.

        Args:
            key (str): The symbolic name to normalize.

        Returns:
            str: The normalized attribute name.

        Example:
            >>> HttpCodeLookup._normalize("continue")
            'continue_'
            >>> HttpCodeLookup._normalize("not_found")
            'not_found'
        """
        if keyword.iskeyword(key):
            return f"{key}_"
        return key

```
Let's create an instance of `HttpCodeLookup` using our http-status code mapping
and see how we can access the values through dot notation.
```commandline
>>> http = HttpCodeLookup(_http_status_codes)
>>> http.ok
200
>>> http.success
200 
>>> http.redirect
302 
>>> http.continue_
100 
>>> http.conflict
409
```

### Country Telephone Calling Codes

International telephone calling codes are numerical prefixes used to identify countries or
numbering regions when placing international calls.

We can represent this information using a Python dictionary, where each country name is 
associated with its corresponding international calling code.

Consider the following mapping, containing a selection of countries and their telephone 
calling codes:

```python
country_dial_codes = {
    "united_states": "+1",
    "united_kingdom": "+44",
    "australia": "+61",
    "india": "+91",
    "china": "+86",
    "japan": "+81",
    "germany": "+49",
    "singapore": "+65",
    "canada": "+1",
    "france": "+33",
    "italy": "+39",
    "spain": "+34",
    "netherlands": "+31",
    "switzerland": "+41",
    "sweden": "+46",
    "norway": "+47",
    "new_zealand": "+64",
    "south_africa": "+27",
    "malaysia": "+60"
}
```
### Building an Attribute-Based Lookup
To access country telephone calling codes using dot notation, we can create a simple class
called CountryDialCodeLookup. The class accepts the country-code mapping during 
initialization and dynamically creates an attribute for each country. The attribute name is
derived from the country name, and its value is set to the corresponding telephone calling
code.
```python
class CountryDialCodeLookup:
    def __init__(self, dial_codes: dict):
        for name, code in dial_codes.items():
            setattr(self, name, code)
```
Let's create an instance of `CountryDialCodeLookup` using our country calling-code mapping
and see how we can access the values through dot notation.
```commandline
>>> dial_code = CountryDialCodeLookup(country_dial_codes)
>>> dial_code.united_states
'+1'
>>> dial_code.india
'+91'
>>> dial_code.japan
'+81'
>>> dial_code.canada
'+1'
```

### Currency Information Mapping
We can extend the same concept to represent currency information associated with different
countries. Unlike the previous example, where each country was associated with a single
calling-code value, this mapping contains two related pieces of information for each country:
the currency name and its three-letter ISO currency code.

We can represent this information using a dictionary, with country names as keys and tuples 
containing the corresponding currency name and code as values.

Consider the following mapping:
```python
currency_info = {
    "india": ("Indian Rupee", "INR"),
    "united_states": ("US Dollar", "USD"),
    "united_kingdom": ("British Pound", "GBP"),
    "japan": ("Japanese Yen", "JPY"),
    "china": ("Chinese Yuan", "CNY"),
    "switzerland": ("Swiss Franc", "CHF"),
    "russia": ("Russian Ruble", "RUB"),
    "malaysia": ("Malaysian Ringgit", "MYR")
}
```
### Creating the Currency Lookup Classes
To expose the currency information through dot notation, we can introduce two classes: 
`CurrencyInfoLookup` and `CountryLookup`.

The `CurrencyInfoLookup` class represents the currency details for an individual country. 
It accepts a tuple containing the currency name and its three-letter code, then creates two
instance attributes: name and code.
```python
class CurrencyInfoLookup:
    def __init__(self, currency_info):
        name = currency_info[0]
        code = currency_info[1]
        setattr(self, "name", name)
        setattr(self, "code", code)
```
The CountryLookup class builds the top-level lookup object. It iterates through the 
country-to-currency mapping and dynamically creates an attribute for each country using 
setattr().

For every country, it creates a `CurrencyInfoLookup` instance and assigns it to the 
corresponding country attribute.

```python
class CountryLookup:
    def __init__(self, codes: dict):
        for country, _codes in codes.items():
            setattr(self, country, CurrencyInfoLookup(_codes))
```
The resulting object allows us to access both currency properties using chained dot notation.

```commandline
>>> currency = CountryLookup(currency_info)
>>> currency.united_states.name
'US Dollar'
>>> currency.united_states.code
'USD'
>>> currency.japan.name
'Japanese Yen'
>>> currency.japan.code
'JPY'
```

### Final Thoughts

In this article, we explored how Python's dynamic attribute creation capabilities can be
used to expose dictionary mappings through a semantic attribute interface.

Using HTTP status codes as our starting point, we created an object that allows numeric status
codes to be accessed through meaningful symbolic names rather than conventional dictionary 
lookups. We then applied the same concept to other practical examples, including international
telephone calling codes and currency information.

These examples demonstrate how different mapping structures can benefit from attribute-based
access. Whether a mapping contains a single value, multiple aliases, or structured information,
we can design a suitable object-oriented interface that makes the underlying data easier
to access and understand.

By using **`setattr`** to create attributes dynamically, we can avoid manually defining individual
attributes for every mapping entry. This also allows us to keep the original mapping data separate
from the interface through which application code accesses it.

However, this technique is best suited to mappings with predictable keys that can be
represented as valid Python attribute names. Considerations such as reserved keywords,
duplicate names, and attribute-name collisions should also be taken into account when
designing a reusable implementation.

The key takeaway is that Python dictionaries do not always have to be accessed through 
dictionary keys. With a small amount of object-oriented abstraction, we can transform 
ordinary mappings into expressive interfaces that make application code easier to read
and work with.
