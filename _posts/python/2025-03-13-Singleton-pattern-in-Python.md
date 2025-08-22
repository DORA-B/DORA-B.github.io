---
title: Singleton Pattern in Python
date: 2025-03-13
categories: [Python]
tags: [python]     # TAG names should always be lowercase
---


[Content Link](https://www.geeksforgeeks.org/singleton-pattern-in-python-a-complete-guide/) 

> Problem: I want to have a global instance of class and access its functions that could be shared within other classes.


Using a singleton pattern has many benefits. A few of them are:

- To limit concurrent access to a shared resource.
- To create a global point of access for a resource.
- To create just one instance of a class, throughout the lifetime of a program.


What is the Factory Pattern in Python with Singleton?
```
The Factory pattern is a creational design pattern used to create objects without specifying the exact class of object that will be created. Combining it with a Singleton can ensure that the factory itself is a singleton or that it produces singletons.
```

# Example of a Factory Pattern with Singleton:
<mark>In this example, car1 is car2, and `car1 == car2` will cause error, because `__eq__` method is not implemented in the `Car` class</mark>

```python
class Singleton:
    _instances = {}

    def __new__(cls, class_name):
        if class_name not in cls._instances:
            cls._instances[class_name] = super(Singleton, cls).__new__(cls)
        return cls._instances[class_name]

class Car:
    pass

class Truck:
    pass

# Factory function
def vehicle_factory(vehicle_type):
    return Singleton(vehicle_type)

# Usage
car1 = vehicle_factory(Car)
car2 = vehicle_factory(Car)
truck1 = vehicle_factory(Truck)

print(car1 is car2)  # Output: True,  both car1 and car2 refer to the same instance of Singleton created for the Car class.
print(car1 is truck1)  # Output: False, because car1 refers to the instance created for the Car class, while truck1 refers to a different instance created for the Truck class.
```

# What is the Difference Between `__new__` and `__init__` ?

```py
class MyClass:
    def __new__(cls):
        print("Creating instance")
        instance = super(MyClass, cls).__new__(cls)
        return instance

    def __init__(self):
        print("Initializing instance")

obj = MyClass()
# Creating instance
# Initializing instance 
```

- `__new__` is a static method responsible for creating a new instance of a class. It is called before `__init__` and is responsible for returning a new instance of your class. In custom object creation scenarios like singletons, `__new__` can be overridden to control how instances are created.
- `__init__` is the constructor method of the class and is called after `__new__` to initialize the new instance. `__init__` doesn’t return anything; it only initializes the object after it’s been created.

# Different ways to implement a Singleton

A singleton pattern can be implemented in three different ways. They are as follows:

- Module-level Singleton
    - Create a shared variable in a python file, and export it as a global one
- Classic Singleton
  ```python
  class SingletonClass(object):
    def __new__(cls):
      if not hasattr(cls, 'instance'):
        cls.instance = super(SingletonClass, cls).__new__(cls)
      return cls.instance
    
  class SingletonChild(SingletonClass):
      pass
    
  singleton = SingletonClass()  
  child = SingletonChild()
  print(child is singleton)

  singleton.singl_variable = "Singleton Variable"
  print(child.singl_variable)
  # True
  # Singleton Variable
  ```
  SingletonChild has the same instance of SingletonClass and also shares the same state. But there are scenarios, where we need a different instance, but should share the same state. This state sharing can be achieved using Borg singleton.
- Borg Singleton: Borg singleton is a design pattern in Python that allows state sharing for different instances. Let’s look into the following code.
  ```py
  class BorgSingleton(object):
    _shared_borg_state = {}
    
    def __new__(cls, *args, **kwargs):
      obj = super(BorgSingleton, cls).__new__(cls, *args, **kwargs)
      obj.__dict__ = cls._shared_borg_state
      return obj
    
  borg = BorgSingleton()
  borg.shared_variable = "Shared Variable"

  class ChildBorg(BorgSingleton):
    pass

  childBorg = ChildBorg()
  print(childBorg is borg)
  print(childBorg.shared_variable)  
  # False. they are different instances
  # Shared Variable, however, share the same variable
  ```

## Explanation of Borg Pattern

The Borg pattern ensures that all instances of a class share the same state (i.e., the same `__dict__`). This is achieved by overriding the `__new__` method and assigning a shared dictionary (`_shared_borg_state`) to the `__dict__` attribute of every instance.

### What Happens When `borg.shared_variable` is Set?
1. When you create an instance of BorgSingleton (`borg = BorgSingleton())`, the `__new__` method is called.

2. Inside `__new__`, the `__dict__` of the new instance (obj) is set to the shared dictionary `_shared_borg_state`.

3. When you set `borg.shared_variable = "Shared Variable"`, you're actually modifying the shared `_shared_borg_state` dictionary, because `borg.__dict__` points to it.

4. So, `shared_variable` is not an attribute of the `BorgSingleton` class itself; it’s a key in the shared `_shared_borg_state` dictionary.

### key ideas

- Even though `borg` and `childBorg` share the same state (`_shared_borg_state`), they are different instances of their respective classes (`BorgSingleton` and `ChildBorg`).

- Shared State: All instances of `BorgSingleton` and its subclasses share the same `_shared_borg_state` dictionary.

- Attribute Access: When you set or access an attribute on an instance (e.g., `borg.shared_variable`), you're actually modifying or reading from the shared `_shared_borg_state` dictionary.

- Different Instances: Even though `borg` and `childBorg` share state, they are different instances, so `childBorg is borg` is False.

### reset `theshared_borg_state` attirbute
If you want a different state, then you can reset the  shared_borg_state attribute. Let’s see how to reset a shared state.

```python
class BorgSingleton(object):
  _shared_borg_state = {}
  
  def __new__(cls, *args, **kwargs):
    obj = super(BorgSingleton, cls).__new__(cls, *args, **kwargs)
    obj.__dict__ = cls._shared_borg_state
    return obj
  
borg = BorgSingleton()
borg.shared_variable = "Shared Variable"

class NewChildBorg(BorgSingleton):
    _shared_borg_state = {} # reset here

newChildBorg = NewChildBorg()
print(newChildBorg.shared_variable) # AttributeError: 'NewChildBorg' object has no attribute 'shared_variable'
```

## Some Understanding based on this

- Shared State: All instances of `BorgSingleton` (and its subclasses) share the same `_shared_borg_state` dictionary. This means that any attribute added to one instance will be accessible to all other instances.

- Instance-Specific Attributes: If a subclass adds its own attributes, those attributes are also stored in the shared `_shared_borg_state` dictionary. This means that all instances (including instances of other subclasses) will see those attributes.

- Class Hierarchy: The Borg pattern does not create separate state dictionaries for subclasses. All instances, regardless of their class in the hierarchy, share the same `_shared_borg_state`.

### Borg pattern does not create a hierarchical state for subclasses, all instances share the same _shared_borg_state

- When you create a new instance of `BorgSingleton` (or its subclasses), the `__dict__` of that instance is set to the shared `_shared_borg_state` dictionary.

- Class variables and functions are treated as references from one single instance (though the class pointers may differ).


If a subclass (e.g., `ChildBorg`) adds its own attributes, those attributes are stored in the shared `_shared_borg_state` dictionary. This means that all instances (including instances of `BorgSingleton` and other subclasses) will see those attributes.

This is indeed "break" the natural class hierarchy in object-oriented programming.
```py
class BorgSingleton:
    _shared_borg_state = {}

    def __new__(cls, *args, **kwargs):
        obj = super(BorgSingleton, cls).__new__(cls, *args, **kwargs)
        obj.__dict__ = cls._shared_borg_state
        return obj

class ChildBorg(BorgSingleton):
    def __init__(self, child_attr):
        self.child_attr = child_attr  # This will be added to the shared state

class GrandchildBorg(ChildBorg):
    def __init__(self, grandchild_attr):
        self.grandchild_attr = grandchild_attr  # This will be added to the shared state


# Create instances
borg = BorgSingleton()
child_borg = ChildBorg("Child Attribute")
grandchild_borg = GrandchildBorg("Grandchild Attribute")

# Access shared state
print(borg.child_attr)  # Output: "Child Attribute" (shared with ChildBorg)
print(child_borg.child_attr)  # Output: "Child Attribute"

# Access shared state
print(borg.grandchild_attr)  # Output: "Grandchild Attribute" (shared with GrandchildBorg)
print(child_borg.grandchild_attr)  # Output: "Grandchild Attribute"
print(grandchild_borg.grandchild_attr)  # Output: "Grandchild Attribute"
```


### Alternatives to create a seperate state for different hirarchy in borg pattern
1. If you want to maintain the class hierarchy while still sharing some state, you can use other design patterns or techniques:

2. Class Variables: Use class variables to share state across instances of a class and its subclasses.

3. Composition Over Inheritance: Use composition to share state between objects instead of relying on inheritance.

4. Hierarchical State: Modify the Borg pattern to use a hierarchical state dictionary, where each class in the hierarchy has its own shared state.

```py
class HierarchicalBorg:
    _shared_borg_states = {}

    def __new__(cls, *args, **kwargs):
        obj = super(HierarchicalBorg, cls).__new__(cls, *args, **kwargs)
        if cls not in cls._shared_borg_states:
            cls._shared_borg_states[cls] = {}
        obj.__dict__ = cls._shared_borg_states[cls]
        return obj

class Parent(HierarchicalBorg):
    pass

class Child(Parent):
    pass

parent = Parent()
parent.parent_attr = "Parent Attribute"

child = Child()
child.child_attr = "Child Attribute"

print(parent.parent_attr)  # Output: "Parent Attribute"
print(child.child_attr)  # Output: "Child Attribute"
print(parent.child_attr)  # AttributeError: 'Parent' object has no attribute 'child_attr'
```