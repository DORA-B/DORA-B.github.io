---
title: Forward reference in Python
date: 2024-03-10
categories: [Python]
tags: [python]     # TAG names should always be lowercase
---
Forward Reference in python
===========================

There is a code snippet like:

```python
@property
def analyses(self) -> "AnalysesHubWithDefault":
    result = self._analyses
    if result is None:
        raise ValueError("Cannot access analyses this early in project lifecycle")
    return result
```

In Python, forward references are addressed using string literals for type hints. This is necessary when you reference a class that is not yet defined at the point where you're using it. This situation often occurs in cases of circular dependencies or when the type hint is for a class that is defined later in the same module.

The reason AnalysesHubWithDefault is enclosed in quotes ("") in the method signature you provided is to indicate that it is a forward reference. This is a way to tell Python: "This name will be defined later, so treat it as the name of a class for type hinting purposes."

Python's type hinting system, introduced in PEP 484, allows for this usage to enable more complex type hints without running into issues with names that are not yet defined. Without the use of quotes, Python would raise a NameError because it tries to evaluate the name as a reference to a class or type object when parsing the function definition. By using quotes, you defer the resolution of the name until later, allowing for the type checker (like mypy or PyCharm's built-in type checker) to resolve the name after all classes have been defined.

As of Python 3.7, with the introduction of the from __future__ import annotations statement (PEP 563), all annotations are stored as string literals by default, which defers the evaluation of type annotations. This means that in Python 3.7 and later, you could write the type hint without quotes if you have the future import at the top of your module:

```python
from __future__ import annotations

def analyses(self) -> AnalysesHubWithDefault:
    ...

```

[Reference](https://peps.python.org/pep-0563/)