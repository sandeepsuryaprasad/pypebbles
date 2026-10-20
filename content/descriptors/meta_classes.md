## Building a Declarative JSON-to-Object Mapper - Metaclass

In the previous article, we implemented a JSON-to-Python object mapping system using a 
class decorator to dynamically attach `Field` descriptors to mapping classes. 
While this approach keeps the implementation simple, Python metaclasses provide 
another way to achieve the same result during class creation.

In this article, we will revisit the same example and replace the class decorator 
with a custom metaclass, `MapFieldsMeta`. The JSON structure, field mappings, and 
overall functionality remain unchanged. The focus is on understanding how a metaclass 
can process the `_nodes` declaration and install the required `Field` descriptors 
automatically when a class is created.

By comparing these two approaches, we can understand how Python metaclasses offer an
alternative mechanism for customizing class creation and centralizing mapping logic.

```python
class MapFieldsMeta(type):
    """Metaclass for dynamically mapping JSON fields to Python attributes.

    The metaclass processes the `_nodes` declaration of a class and
    dynamically attaches `Field` descriptors for the specified JSON fields.
    It also provides an initializer that populates the instance dictionary
    with the supplied JSON data.

    Classes using this metaclass can declare their field mappings through
    `_nodes`, where each entry specifies a field name and its associated
    mapping type. A mapping type of `None` represents a direct field mapping,
    while a class reference identifies the mapping used for nested data.

    During class creation, the metaclass adds the initializer and constructs
    the corresponding `Field` descriptors before delegating class creation
    to the parent `type` implementation.

    This approach centralizes the mapping logic in the metaclass, eliminating
    the need to apply a separate class decorator to every mapping class.
    """
    def __new__(mcls, clsname, bases, clsdict):
        """Create a class with dynamically generated field descriptors.

        Processes the class namespace before the class object is created.
        An initializer is added to populate instance attributes from the
        supplied information dictionary, and a `Field` descriptor is
        installed for each entry declared in `_nodes`.

        Args:
            mcls (type): The metaclass responsible for creating the class.
            clsname (str): The name of the class being created.
            bases (tuple): The base classes inherited by the new class.
            clsdict (dict): The namespace containing the attributes defined
                in the class body, including the optional `_nodes` mapping.

        Returns:
            type: The newly created class with its initializer and field
                descriptors installed.

        Notes:
            Each `_nodes` entry is a tuple containing a field name and its
            associated mapping type. The descriptors are added to the class
            namespace before `type.__new__` creates the class object.

            The initializer assumes that the supplied information is a
            dictionary and updates the instance dictionary directly.
            Consequently, descriptor setters are not invoked by this update.
        """
        clsdict["__init__"] = lambda self, info: self.__dict__.update(info)

        nodes = clsdict.get("_nodes")
        for node, mapping in nodes:
            clsdict[node] = Field(node, mapping)

        return super().__new__(mcls, clsname, bases, clsdict)
```
### Understanding MapFieldsMeta
The `MapFieldsMeta` class is a custom Python metaclass that automates the creation of 
mapping classes. It processes a class's `_nodes` declaration, dynamically creates `Field` 
descriptors for the specified JSON fields, and provides a common initializer for mapping
objects.

In the previous implementation, a class decorator performed these operations after the
class had been created. With a metaclass, the same operations can be performed **during 
class** creation, allowing the mapping logic to be centralized in a single place.

Let's examine the implementation step by step,

#### Defining a Custom Metaclass
```python
class MapFieldsMeta(type)
```

By inheriting from `type`, `MapFieldsMeta` customizes how Python creates classes 
that use it.

```python
class Contact(metaclass=MapFieldsMeta):
    _nodes = [
        ("email", None),
        ("phone", None),
    ]
```
When Python executes this class definition, it invokes `MapFieldsMeta` class's `__new__` 
to construct the `Contact` class. Before the class is created, the metaclass can inspect 
its declared attributes and modify the class namespace.

This makes a metaclass suitable for processing declarative mappings such as `_nodes`, 
where developers specify what should be mapped rather than writing the mapping 
logic repeatedly.

#### Understanding the `__new__` Method
```python
def __new__(mcls, clsname, bases, clsdict):
```

The `__new__` method is the central part of the metaclass implementation. 
It is responsible for creating the class object.

Python calls this method with four arguments:

- **mcls** — The metaclass itself, which is MapFieldsMeta in this example.
- **clsname** — The name of the class being created, such as `Contact`.
- **bases** — A tuple containing the class's base classes.
- **clsdict** — The namespace containing the attributes defined in the class body, 
including `_nodes`.

By the time `MapFieldsMeta.__new__` is invoked, Python has already executed the class 
body and collected its definitions into `clsdict`. However, the final class object 
has not yet been created.

This provides an opportunity to inspect and modify the namespace before passing it 
to `type.__new__`.

#### Injecting the initializer
```python
clsdict["__init__"] = lambda self, info: self.__dict__.update(info)
```

The metaclass adds an `__init__` method to the class namespace. 
This initializer accepts an `info` dictionary and copies its entries into the instance's
attribute dictionary.

Example,
```python
contact = Contact({
    "email": "john.doe@example.com",
    "phone": "+1-123-555-0987",
})
```
When `Contact` is instantiated, the generated initializer stores the supplied values in
`contact.__dict__`. Consequently, the instance contains attributes such as `email` and `phone`.

Because the initializer is injected by the metaclass, individual mapping classes do not
need to define their own repetitive initialization methods.

#### Processing the _nodes Declaration
```python
nodes = clsdict.get("_nodes")
for node, mapping in nodes:
    clsdict[node] = Field(node, mapping)
```
The `_nodes` attribute describes how individual JSON fields should be mapped to 
Python attributes.

Each entry is a tuple containing,
- **node** The JSON field name.
- **maping** The mapping type associated with that field.

A mapping type of `None` represents a direct field mapping or a scalar value, 
while a class reference identifies the mapping class used for nested data.

Consider the following declaration:
```python
class Contact(metaclass=MapFieldsMeta):
    _nodes = [
        ("email", None),
        ("phone", None),
    ]
```
The metaclass retrieves `_nodes` from `clsdict` and iterates through its entries. 
For each entry, it creates a `Field` descriptor and inserts it into the namespace
under the corresponding field name.

Conceptually, the namespace is modified to include:
```python
clsdict["email"] = Field("email", None)
clsdict["phone"] = Field("phone", None)
```
These assignments do not directly store the JSON values. Instead, they create instances of 
descriptors on the class that can implement the field-mapping behavior defined by `Field`.

For a nested mapping, the declaration might look like this,

```python
class Passenger(metaclass=MapFieldsMeta):
    _nodes = [
        ("passenger_id", None),
        ("first_name", None),
        ("contact", Contact),
    ]
```

Here, `contact` is associated with the `Contact` mapping class. 
The `Field` descriptor can use that mapping information to construct or expose a
nested `Contact` object, depending on how the descriptor is implemented.

#### Creating the Class Object
```python
return super().__new__(mcls, clsname, bases, clsdict)
```

After modifying the namespace, the metaclass delegates the actual class creation 
to its parent implementation, `type.__new__`.

The arguments are passed along unchanged, except that `clsdict` now contains the 
generated initializer and the `Field` descriptors.

This process allows each mapping class to declare its fields while leaving the 
repetitive class-construction logic to the metaclass.

#### Final Thoughts
`MapFieldsMeta` demonstrates how Python metaclasses can be used to build declarative 
object-mapping systems. By processing `_nodes` during class creation, the metaclass 
automatically create field descriptors and a common initializer, eliminating the
need to repeat this setup in every mapping class.

Compared with the class decorator implementation from the previous article, 
the overall mapping functionality remains the same, but the mechanism for creating
the mapping changes. A class decorator modifies a class after it has been created, 
whereas a metaclass can modify the namespace before the class object is created.

Both approaches are useful forms of Python metaprogramming. The choice depends on
where the customization belongs and how much control is needed over class creation.
For this example, the metaclass provides a centralized way to process mapping 
declarations and demonstrates how Python can use its own class-construction mechanism
to reduce repetitive code.