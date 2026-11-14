[Previous Article](../profiling/profiling_memory.md)

## Implementing the Result Object Pattern for Robust Python CLI Validation

Command-line applications often require validating user-supplied inputs before executing their primary operations.
These inputs may include file paths, configuration options, sorting criteria, or other application-specific parameters.
Handling invalid inputs cleanly is essential to building reliable and user-friendly CLI applications.

A common approach is to use conditional checks or exceptions to handle validation failures. However, as the number of
validation rules grows, managing these outcomes can make the application logic increasingly difficult to maintain.

In this article, we explore the **Result Object Pattern** and how it can simplify input validation in Python
CLI applications. Using **fsort** a Python command-line utility for sorting files, we will introduce a lightweight
**Result** class that explicitly represents the outcome of validation operations.

We will see how this approach separates validation logic from command-line error handling, makes success and failure
outcomes explicit, and helps keep the application's execution flow clean and maintainable.

### Introducing fsort

fsort is a Python command-line utility that sorts files within a specified directory based on different criteria,
such as file name, size, or modification date. It also supports filtering files using extension patterns.

Like most CLI applications, fsort accepts user-supplied arguments that determine how the application should execute.
Before starting the sorting operation, these inputs must be validated to ensure that the specified directory exists,
the sorting criterion is supported, and the application receives valid values.

### Understanding the fsort Implementation

Before introducing the Result Object Pattern, let's briefly look at how fsort is structured 
and how its execution flows.

The library has two important responsibilities: Command-line orchestration: The main() 
function parses the command-line arguments and coordinates the execution of the application.

File sorting: The FileSort class implements the actual sorting functionality, organizing 
files based on the selected sorting criterion.

### The main() Function

The **main()** function acts as the entry point of the CLI application. It uses Python's 
argparse module to process command-line arguments such as the directory path, sorting 
criterion, and file extension pattern.

Once the arguments are parsed, the function passes the corresponding values to the sorting
implementation.

```python
def main(argv= None) -> int:
    """Executes sort"""
    parser = _parser()
    args = parser.parse_args(argv)
    file = FileSort()
    file.sort(path=args.path, by_what=args.key, pattern=args.extension)
    return 0
```

### The File Sorting Implementation

The `FileSort` class is responsible for performing the actual sorting operation. 
It retrieves files from the specified directory and organizes them according to the requested 
sorting criterion, such as file name, size, or modification date.

Keeping the sorting functionality separate from command-line argument handling allows each 
component to focus on a specific responsibility.

```python
from dataclasses import dataclass
from datetime import datetime
from pathlib import Path
from time import sleep
from typing import Optional


@dataclass
class FileInfo:
    name: str
    date: datetime
    size: float


class FileSort:
    def _to_mega_bytes(self, bytes: float) -> float:
        return round(bytes / 1024**2, 2)

    def _stats(self, path: Path) -> tuple:
        stats = path.stat()
        file_name = path.name
        last_modified = datetime.fromtimestamp(stats.st_mtime)
        file_size = self._to_mega_bytes(stats.st_size)
        return (file_name, last_modified, file_size)


    def _files_to_sort(self, path: str, extension: str = "*") -> Optional[list[FileInfo]]:
        """Returns list of FileInfo objects"""
        path = Path(path)
        files: list[FileInfo] = []
        for item in path.glob(f"*.{extension}"):
            if item.is_file():
                name, last_modified, size = self._stats(item)
                file_info = FileInfo(name, last_modified, size)
                files.append(file_info)
        return files

    def _print_results(self, items: list[FileInfo]) -> None:
        """Prints the results in the console"""
        for item in items:
            sleep(0.05)
            # Converting all fields to str
            _last_modified: str = item.date.strftime("%Y-%m-%d %H:%M:%S")
            _size: str = f"{item.size:.2f} MB"
            _file_name: str = item.name
            print(f"{_last_modified:<20} {_size:>14}\t{_file_name}")

    def sort(self, path: str, by_what: str, pattern: str) -> None:
        """Sorts the list of files"""
        files_to_sort = self._files_to_sort(path, pattern)
        if not files_to_sort:
            return
        self._print_results(sorted(files_to_sort, key=lambda item: getattr(item, by_what)))
```
### Where Does Input Validation Fit In?
Although the existing implementation handles argument parsing and file sorting, 
user-supplied inputs must be validated before the sorting operation begins.
For example:

* Does the specified directory exist?
* Is the supplied path actually a directory?
* Is the requested sorting criterion supported?

These checks are essential to prevent invalid inputs from reaching the sorting implementation.

This is where we can improve the existing design by introducing the Result Object Pattern. 
Before invoking FileSort, we will validate the inputs and explicitly represent the outcome 
of each validation operation using a Result object.

### Introducing Result Object
```python
class Result:
    """Represent the outcome of an operation.

    Stores either a successful result value or an error describing
    why the operation failed.

    Attributes:
        value: The value returned by a successful operation.
        error: An error message if the operation failed.

    The `is_ok` property indicates whether the operation succeeded.
    """

    def __init__(self, *, value=None, error=None):
        self.value = value
        self.error = error

    @property
    def is_ok(self):
        """Indicate whether the operation completed successfully."""
        return self.error is None

class Validate:
    def validate_path(self, path: str):
        """Validate that the given path exists and is a directory.

        Args:
            path: The directory path to validate.

        Returns:
            Result: Contains the validated Path object on success,
                or an error message if validation fails.
        """
        path = Path(path)
        if not path.exists():
            return Result(error=f"{path!r} does not exist")
        if not path.is_dir():
            return Result(error=f"{path!r} does not exist")
        return Result()

    def validate_key(self, key: str):
        """Validate the sorting key against the supported options.

        Args:
            key: The sorting key to validate.

        Returns:
            Result: Contains the validated key on success,
                or an error message if the key is invalid.
        """
        sorting_choices = ("name", "size", "date")
        if key not in sorting_choices:
            return Result(error=f"{key!r} is invalid key")
        return Result()
```
The Result class provides a simple way to represent the outcome of an operation. 
It encapsulates either a successful result or an error message, using the `is_ok` property 
to indicate whether the operation completed successfully.

The Validate class uses this approach to implement input validation independently of the 
command-line interface. The `validate_path` method checks whether the supplied path 
exists and is a directory, while `validate_key` verifies that the sorting key is one of the
supported options: `name`, `size`, or `date`.

When validation fails, the methods return a `Result` object containing an appropriate error message.
When validation succeeds, they return a `Result` object indicating success. This allows the 
calling code to handle validation outcomes consistently without relying on exceptions 
for expected validation failures.

### Modified main() function

```python
def main(argv=None) -> int:
    """Executes sort"""
    parser = _parser()
    args = parser.parse_args(argv)

    validate = Validate()

    path = args.path if args.path else "."
    result =  validate.validate_path(path)
    if not result.is_ok:
        parser.error(result.error)

    key = args.key if args.key else "name"
    result = validate.validate_key(key)
    if not result.is_ok:
        parser.error(result.error)

    extension = args.extension if args.extension else "*"

    file = FileSort()
    file.sort(path=path, by_what=key, pattern=extension)
    return 0
```
### Why Separate Validation into a Validate Class?
Keeping validation logic in a dedicated Validate class provides a clear separation of 
responsibilities. Instead of embedding input checks directly into the main() function, 
each validation rule is implemented as an independent method.

This offers several advantages:

* **Separation of concerns**: The main() function focuses on coordinating application execution,
while the Validate class handles input validation.

* **Reusability**: Validation methods can be reused wherever the same inputs need to be
validated, without duplicating the checks.

* **Maintainability**: New validation rules can be introduced or existing ones modified without
making the main application flow increasingly complex.

By keeping validation logic independent of argparse and the application's sorting 
implementation, we create a cleaner structure that is easier to extend, test, and maintain.

### Full Implementation
The implementation presented in this article is a simplified version to explain the core 
concepts clearly. The complete **`fsort`** library includes additional validation,
error handling, tests, and other improvements required for a reusable open-source Python
library. 
You can explore the complete implementation and contribute to the project on **[GitHub](https://github.com/sandeepsuryaprasad/fsort)**

### Final Thoughts
In this article, we explored the Result Object Pattern as a way to handle expected validation 
failures in Python command-line applications.

Using fsort as an example, we introduced a lightweight Result class to represent the outcome 
of validation operations explicitly. The Validate class uses this object to report whether 
an input is valid or provide an error message explaining why validation failed.

This approach keeps validation logic independent of the command-line interface while
allowing main() to handle failures consistently. It also avoids using exceptions for
expected validation scenarios, making the application's execution flow easier to follow.

The Result Object Pattern is not intended to replace exceptions entirely. Exceptions remain
appropriate for unexpected failures, while a Result object provides a structured way to 
communicate outcomes that are part of an operation's normal behaviour.

By introducing this small abstraction, we can make validation code more predictable, readable,
and easier to maintain as a Python CLI application grows.

[Previous Article](../profiling/profiling_memory.md)