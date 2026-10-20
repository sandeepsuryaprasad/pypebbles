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