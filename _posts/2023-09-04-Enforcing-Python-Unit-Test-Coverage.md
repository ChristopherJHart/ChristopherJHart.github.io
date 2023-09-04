---
layout: post
title: Granular Enforcement of Python Unit Test Coverage through Code Inspection
---

If you're maintaining a medium-sized software project, you've probably found yourself in a situation where you've added a new feature or model to your Python project and then realized that you forgot to write unit tests for it. You might have code coverage tools in place, but measuring code coverage of unit tests is an imperfect science that can sometimes give a false sense of security. We can supplement code coverage tools by enforcing unit test coverage through "tests for our tests". Python's "everything is an object" philosophy makes it easy for us to detect when new code is added and validate whether one or more unit tests exist for it.

If you're interested, the source code for this proof of concept can be found in the [GitHub repository here](https://github.com/ChristopherJHart/enforcing-unit-tests).

## Project Structure

Let's start by taking a look at our project structure. Fire up your terminal and run the `tree` command:

```
(venv) christopher@ubuntu-playground:~/GitHub/enforcing-unit-tests$ tree ./ -P *.py -I "*.pyc|venv/|__pycache__"
./
├── project
│   ├── __init__.py
│   └── models.py
└── tests
    ├── __init__.py
    ├── test_models_dynamic.py
    └── test_models_static.py

2 directories, 5 files
```

The `project` directory houses our Python software application, including a set of data models in `project/models.py`. The contents of this file are shown below:

```python
"""Houses models that are used elsewhere in the project.

All of these models are imported into the test_models.py file for testing.
"""

NETWORK_VRFS = [
    "red",
    "blue",
    "green",
]

IPV4_SUBNETS = [
    "192.168.0.0/24",
    "192.168.1.0/24",
    "10.250.250.0/24",
]
```

We have some simple unit tests for our models defined in `tests/test_models_static.py`. Here, we are importing all models from `project/models.py` and testing whether they are lists of strings.

```python
"""Houses static unit tests for the models.py file."""

from typing import List

from project import models


def generic_list_of_strings_test(list_of_strings: List[str]) -> None:
    """Tests whether a supposed list of strings is as described."""
    assert isinstance(list_of_strings, list)
    assert len(list_of_strings) > 0
    for item in list_of_strings:
        assert isinstance(item, str)


def test_NETWORK_VRFS() -> None:
    """Ensure that the NETWORK_VRFS model is a list of strings."""
    generic_list_of_strings_test(models.NETWORK_VRFS)


def test_IPV4_SUBNETS() -> None:
    """Ensure that the IPV4_SUBNETS model is a list of strings."""
    generic_list_of_strings_test(models.IPV4_SUBNETS)
```

## The Problem

If we run these unit tests with code coverage reporting enabled, we can see that they pass with 100% test coverage:

```
(venv) christopher@ubuntu-playground:~/GitHub/enforcing-unit-tests$ python -m pytest --cov=project tests/test_models_static.py 
=========================================== test session starts ============================================
platform linux -- Python 3.10.12, pytest-7.4.1, pluggy-1.3.0
rootdir: /home/christopher/GitHub/enforcing-unit-tests
plugins: cov-4.1.0
collected 2 items                                                                                          

tests/test_models_static.py ..                                                                       [100%]

---------- coverage: platform linux, python 3.10.12-final-0 ----------
Name                  Stmts   Miss  Cover
-----------------------------------------
project/__init__.py       0      0   100%
project/models.py         2      0   100%
-----------------------------------------
TOTAL                     2      0   100%


============================================ 2 passed in 0.02s =============================================
```

Next, let's add a new model, `IPV6_SUBNETS`, to `project/models.py`.

```python
"""Houses models that are used elsewhere in the project.

All of these models are imported into the test_models.py file for testing.
"""

NETWORK_VRFS = [
    "red",
    "blue",
    "green",
]

IPV4_SUBNETS = [
    "192.168.0.0/24",
    "192.168.1.0/24",
    "10.250.250.0/24",
]

IPV6_SUBNETS = [
    "2001:db8:1::/64",
    "2001:db8:2::/64",
    "2001:db8:3::/64",
]
```

If we re-run these unit tests with coverage reporting enabled, we can see that they still have 100% coverage. This is not correct, since we did not add a third unit test for the new `IPV6_SUBNETS` model:

```
(venv) christopher@ubuntu-playground:~/GitHub/enforcing-unit-tests$ python -m pytest --cov=project tests/test_models_static.py 
====================== test session starts ======================
platform linux -- Python 3.10.12, pytest-7.4.1, pluggy-1.3.0
rootdir: /home/christopher/GitHub/enforcing-unit-tests
plugins: cov-4.1.0
collected 2 items                                               

tests/test_models_static.py ..                            [100%]

---------- coverage: platform linux, python 3.10.12-final-0 ----------
Name                  Stmts   Miss  Cover
-----------------------------------------
project/__init__.py       0      0   100%
project/models.py         3      0   100%
-----------------------------------------
TOTAL                     3      0   100%


======================= 2 passed in 0.02s =======================
(venv) christopher@ubuntu-playground:~/GitHub/enforcing-unit-test
```

**This is the crux of the problem** - we have added new code, *forgot to add unit tests for it*, and our code coverage tool failed to catch the gap.

If this project is a proper software development project where multiple developers are reviewing each other's code, it's possible that during the code review process, a fellow developer could catch that this change did not include unit tests for the new model. However, if this project is backed by a single developer, or reviewers are suffering from [code review fatigue](https://tylercipriani.com/blog/2022/03/12/code-review-procrastination-and-clarity/), it's possible for missing unit tests to slip through the cracks. And since coverage tools may not catch the coverage gap, nobody would be the wiser.

## Programmatically Enforcing Unit Test Coverage

To fix this, we can enforce code coverage against our models through code inspection using the `inspect` module. Let's start building these unit tests out in a new file called `tests/test_models_dynamic.py`, starting with a function called `test_all_models_have_unit_tests()`:

```python
"""Houses dynamic unit tests for the models.py file."""

import inspect

import pytest

from project import models

def test_all_models_have_unit_tests() -> None:
    """Unit test that validates test coverage for models."""
    model_objects = inspect.getmembers(models)
    for model_object, _ in model_objects:
        test_name = f"test_{model_object}"
        for global_object in globals():
            if test_name in global_object:
                break
        else:
            pytest.fail(f"Missing unit test for model {model_object}.")
```

This function uses the `inspect.get_members()` function to pull a list of all objects in the `project.models` module. Then, it iterates through each object and checks whether there is a unit test defined in the current `tests/test_models_dynamic.py` file that corresponds with the object. If there is not, it fails the test.

If we run our unit tests now, we can see that this unit test fails because we have not defined a unit test for the `IPV4_SUBNETS` model.

```
(venv) christopher@ubuntu-playground:~/GitHub/enforcing-unit-tests$ python -m pytest tests/test_models_dynamic.py 
================================== test session starts ==================================
platform linux -- Python 3.10.12, pytest-7.4.1, pluggy-1.3.0
rootdir: /home/christopher/GitHub/enforcing-unit-tests
plugins: cov-4.1.0
collected 1 item                                                                        

tests/test_models_dynamic.py F                                                    [100%]

======================================= FAILURES ========================================
____________________________ test_all_models_have_unit_tests ____________________________

    def test_all_models_have_unit_tests() -> None:
        """Unit test that validates test coverage for models."""
        model_objects = inspect.getmembers(models)
        for model_object, _ in model_objects:
            test_name = f"test_{model_object}"
            for global_object in globals():
                if test_name in global_object:
                    break
            else:
>               pytest.fail(f"Missing unit test for model {model_object}.")
E               Failed: Missing unit test for model IPV4_SUBNETS.

tests/test_models_dynamic.py:29: Failed
================================ short test summary info ================================
FAILED tests/test_models_dynamic.py::test_all_models_have_unit_tests - Failed: Missing unit test for model IPV4_SUBNETS.
=================================== 1 failed in 0.03s ===================================
```

Let's add our unit tests for the `IPV4_SUBNETS` and `NETWORK_VRFS` models:

```python
"""Houses dynamic unit tests for the models.py file."""

import inspect
from typing import List

import pytest

from project import models

def test_all_models_have_unit_tests() -> None:
    """Unit test that validates test coverage for models."""
    model_objects = inspect.getmembers(models)
    for model_object, _ in model_objects:
        test_name = f"test_{model_object}"
        for global_object in globals():
            if test_name in global_object:
                break
        else:
            pytest.fail(f"Missing unit test for model {model_object}.")


def generic_list_of_strings_test(list_of_strings: List[str]) -> None:
    """Tests whether a supposed list of strings is as described."""
    assert isinstance(list_of_strings, list)
    assert len(list_of_strings) > 0
    for item in list_of_strings:
        assert isinstance(item, str)


def test_NETWORK_VRFS() -> None:
    """Ensure that the NETWORK_VRFS model is a list of strings."""
    generic_list_of_strings_test(models.NETWORK_VRFS)


def test_IPV4_SUBNETS() -> None:
    """Ensure that the IPV4_SUBNETS model is a list of strings."""
    generic_list_of_strings_test(models.IPV4_SUBNETS)
```

Let's re-run our unit tests - this time, the unit tests fail because we have not defined a unit test for the `IPV6_SUBNETS` model. However, the two unit tests we've defined for the `NETWORK_VRFS` and `IPV4_SUBNETS` models do pass.

```
(venv) christopher@ubuntu-playground:~/GitHub/enforcing-unit-tests$ python -m pytest tests/test_models_dynamic.py 
================================== test session starts ==================================
platform linux -- Python 3.10.12, pytest-7.4.1, pluggy-1.3.0
rootdir: /home/christopher/GitHub/enforcing-unit-tests
plugins: cov-4.1.0
collected 3 items                                                                       

tests/test_models_dynamic.py F..                                                  [100%]

======================================= FAILURES ========================================
____________________________ test_all_models_have_unit_tests ____________________________

    def test_all_models_have_unit_tests() -> None:
        """Unit test that validates test coverage for models."""
        model_objects = inspect.getmembers(models)
        for model_object, _ in model_objects:
            test_name = f"test_{model_object}"
            for global_object in globals():
                if test_name in global_object:
                    break
            else:
>               pytest.fail(f"Missing unit test for model {model_object}.")
E               Failed: Missing unit test for model IPV6_SUBNETS.

tests/test_models_dynamic.py:29: Failed
================================ short test summary info ================================
FAILED tests/test_models_dynamic.py::test_all_models_have_unit_tests - Failed: Missing unit test for model IPV6_SUBNETS.
============================== 1 failed, 2 passed in 0.03s ==============================
```

Let's add a third unit test for the `IPV6_SUBNETS` model:

```python
"""Houses dynamic unit tests for the models.py file."""

import inspect
from typing import List

import pytest

from project import models


def test_all_models_have_unit_tests() -> None:
    """Unit test that validates test coverage for models."""
    model_objects = inspect.getmembers(models)
    for model_object, _ in model_objects:
        test_name = f"test_{model_object}"
        for global_object in globals():
            if test_name in global_object:
                break
        else:
            pytest.fail(f"Missing unit test for model {model_object}.")


def generic_list_of_strings_test(list_of_strings: List[str]) -> None:
    """Tests whether a supposed list of strings is as described."""
    assert isinstance(list_of_strings, list)
    assert len(list_of_strings) > 0
    for item in list_of_strings:
        assert isinstance(item, str)


def test_NETWORK_VRFS() -> None:
    """Ensure that the NETWORK_VRFS model is a list of strings."""
    generic_list_of_strings_test(models.NETWORK_VRFS)


def test_IPV4_SUBNETS() -> None:
    """Ensure that the IPV4_SUBNETS model is a list of strings."""
    generic_list_of_strings_test(models.IPV4_SUBNETS)


def test_IPV6_SUBNETS() -> None:
    """Ensure that the IPV6_SUBNETS model is a list of strings."""
    generic_list_of_strings_test(models.IPV6_SUBNETS)

```

Let's re-run our unit tests. This time, we see the `test_all_models_have_unit_tests()` unit test fails because of the `__builtins__` object imported from the `project.models` module, which most Python modules will have defined "under the hood". For our purposes, we don't care about this object, so we need to refactor the `test_all_models_have_unit_tests()` function to do two things:

1. Filter the imported objects through a [*predicate function*](https://docs.python.org/3/howto/functional.html#built-in-functions). The `projects.models` module may contain helper functions or classes that are unrelated to the models we want to test. To work around this, we will define a predicate function called `is_builtin_data_structure_or_type()` that will return `True` if the object is a built-in data structure or type, and `False` otherwise. Then, we will pass this predicate function to the `inspect.getmembers()` function as the second argument.
2. Within the `test_all_models_have_unit_tests()` function, we will check to see if objects returned by the `inspect.getmembers()` function are a "dunder" method or attribute starting with `__`. If they are, we will ignore them.

First, the new `is_builtin_data_structure_or_type()` predicate function is shown below.

```python
def is_builtin_data_structure_or_type(obj: object) -> bool:
    """Return True if obj is a builtin data structure or type."""
    return isinstance(obj, (list, dict, set, tuple, bool, bytes, float, int, str))
```

Next, the refactored `test_all_models_have_unit_tests()` function is shown below.

```python
def test_all_models_have_unit_tests() -> None:
    """Unit test that validates test coverage for models."""
    model_objects = inspect.getmembers(models, is_builtin_data_structure_or_type)
    for model_object, _ in model_objects:
        if model_object.startswith("__"):
            print(f"Ignoring magic method or attribute {model_object}")
            continue
        test_name = f"test_{model_object}"
        for global_object in globals():
            if test_name in global_object:
                break
        else:
            pytest.fail(f"Missing unit test for model {model_object}.")
```

Our full `tests/test_models_dynamic.py` file is shown below.

```python
"""Houses dynamic unit tests for the models.py file."""

import inspect
from typing import List

import pytest

from project import models


def is_builtin_data_structure_or_type(obj: object) -> bool:
    """Return True if obj is a builtin data structure or type."""
    return isinstance(obj, (list, dict, set, tuple, bool, bytes, float, int, str))


def test_all_models_have_unit_tests() -> None:
    """Unit test that validates test coverage for models."""
    model_objects = inspect.getmembers(models, is_builtin_data_structure_or_type)
    for model_object, _ in model_objects:
        if model_object.startswith("__"):
            print(f"Ignoring magic method or attribute {model_object}")
            continue
        test_name = f"test_{model_object}"
        for global_object in globals():
            if test_name in global_object:
                break
        else:
            pytest.fail(f"Missing unit test for model {model_object}.")


def generic_list_of_strings_test(list_of_strings: List[str]) -> None:
    """Tests whether a supposed list of strings is as described."""
    assert isinstance(list_of_strings, list)
    assert len(list_of_strings) > 0
    for item in list_of_strings:
        assert isinstance(item, str)


def test_NETWORK_VRFS() -> None:
    """Ensure that the NETWORK_VRFS model is a list of strings."""
    generic_list_of_strings_test(models.NETWORK_VRFS)


def test_IPV4_SUBNETS() -> None:
    """Ensure that the IPV4_SUBNETS model is a list of strings."""
    generic_list_of_strings_test(models.IPV4_SUBNETS)


def test_IPV6_SUBNETS() -> None:
    """Ensure that the IPV6_SUBNETS model is a list of strings."""
    generic_list_of_strings_test(models.IPV6_SUBNETS)
```

Let's re-run our unit tests and confirm that all of our unit tests are now passing:

```
(venv) christopher@ubuntu-playground:~/GitHub/enforcing-unit-tests$ python -m pytest tests/test_models_dynamic.py 
================================== test session starts ==================================
platform linux -- Python 3.10.12, pytest-7.4.1, pluggy-1.3.0
rootdir: /home/christopher/GitHub/enforcing-unit-tests
plugins: cov-4.1.0
collected 4 items                                                                       

tests/test_models_dynamic.py ....                                                 [100%]

=================================== 4 passed in 0.01s ===================================
```

## Conclusion

Now, we have a way to enforce that all of our models have unit tests defined for them. If we add a new model to our `project/models.py` file, we will be notified that we need to add a unit test for it. This is a great way to supplement code coverage tools and ensure that we have unit tests for all of our models. Furthermore, we can standardize the names of unit tests for our models, which makes it easier for developers to understand what a unit test does. This is a purposefully simple example, but applies well to classes, functions, and other objects that we want to ensure have unit tests defined for them.
