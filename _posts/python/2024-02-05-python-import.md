---
title: Python import
date: 2024-02-05
categories: [Python]
tags: [python]     # TAG names should always be lowercase
---
# python import


[Reference](https://realpython.com/python-import/)

```txt
world/
│
├── africa/
│   ├── __init__.py
│   └── zimbabwe.py
│
├── europe/
│   ├── __init__.py
│   ├── greece.py
│   ├── norway.py
│   └── spain.py
│
└── __init__.py
```

in this case, if every country file prints out a greeting, when `__init__.py` files selectively import some of the subpackages and submodules:

```py
# world/africa/__init__.py  (Empty file)

# world/africa/zimbabwe.py
print("Shona: Mhoroyi vhanu vese")
print("Ndebele: Sabona mhlaba")

# world/europe/__init__.py
from . import greece
from . import norway

# world/europe/greece.py
print("Greek: Γειά σας Κόσμε")

# world/europe/norway.py
print("Norwegian: Hei verden")

# world/europe/spain.py
print("Castellano: Hola mundo")

# world/__init__.py
from . import africa
```

Note that `world/__init__.py` imports only `africa` and not `europe`. Similarly, `world/africa/__init__.py` doesn’t import anything, while `world/europe/__init__.py` imports `greece` and `norway` but not `spain`. Each country module will print a greeting when it’s imported.

```python
>>> import world
>>> world
<module 'world' from 'world/__init__.py'>

>>> # The africa subpackage has been automatically imported
>>> world.africa
<module 'world.africa' from 'world/africa/__init__.py'>

>>> # The europe subpackage has not been imported
>>> world.europe
AttributeError: module 'world' has no attribute 'europe'
```

When `europe` is imported, the `europe.greece` and `europe.norway` modules are imported as well. You can see this because the country modules print a greeting when they’re imported:

```python
>>> # Import europe explicitly
>>> from world import europe
Greek: Γειά σας Κόσμε
Norwegian: Hei verden

>>> # The greece submodule has been automatically imported
>>> europe.greece
<module 'world.europe.greece' from 'world/europe/greece.py'>

>>> # Because world is imported, europe is also found in the world namespace
>>> world.europe.norway
<module 'world.europe.norway' from 'world/europe/norway.py'>

>>> # The spain submodule has not been imported
>>> europe.spain
AttributeError: module 'world.europe' has no attribute 'spain'

>>> # Import spain explicitly inside the world namespace
>>> import world.europe.spain
Castellano: Hola mundo

>>> # Note that spain is also available directly inside the europe namespace
>>> europe.spain
<module 'world.europe.spain' from 'world/europe/spain.py'>

>>> # Importing norway doesn't do the import again (no output), but adds
>>> # norway to the global namespace
>>> from world.europe import norway
>>> norway
<module 'world.europe.norway' from 'world/europe/norway.py'>
```

The `world/africa/__init__.py` file is empty. This means that importing the `world.africa` package creates the namespace but has no other effect:

```python
>>> # Even though africa has been imported, zimbabwe has not
>>> world.africa.zimbabwe
AttributeError: module 'world.africa' has no attribute 'zimbabwe'

>>> # Import zimbabwe explicitly into the global namespace
>>> from world.africa import zimbabwe
Shona: Mhoroyi vhanu vese
Ndebele: Sabona mhlaba

>>> # The zimbabwe submodule is now available
>>> zimbabwe
<module 'world.africa.zimbabwe' from 'world/africa/zimbabwe.py'>

>>> # Note that zimbabwe can also be reached through the africa subpackage
>>> world.africa.zimbabwe
<module 'world.africa.zimbabwe' from 'world/africa/zimbabwe.py'>
```

when executing python modules:

> using `python -m [submodules]` to solve the import issues
