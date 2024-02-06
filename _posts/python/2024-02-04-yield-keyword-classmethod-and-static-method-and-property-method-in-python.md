---
title: Yield Keyword Classmethod and static method and property method in python
date: 2024-02-04
categories: [Python]
tags: [python]     # TAG names should always be lowercase
---
[Reference](https://elfi-y.medium.com/python-properties-and-class-methods-a6c7ad69b0f1)

# What is Yield Keyword in Python

Python's yield keyword is like another one we use to return an expression or object, typically in functions, called return. There is a small amount of fluctuation, though. The yield statement of a function returns a generator object rather than just returning a value to the call of the function that contains the statement.

The Python interpreter halts the execution of the function when a yield statement is encountered when calling a function in a program. The generator class returns an object to the caller.

To put it another way, any expression that is passed to the yield keyword will be converted into a generator object, which will subsequently be returned to the caller. As a result, to acquire the values, we must repeatedly access the generator object.

```python
def generator_func():  
    yield "Yield"  
    yield "Keyword"  
    yield "in"  
    yield "Python"  
# constructing a generator object and calling the generator function  
generator_object = generator_func()  
print( type(generator_object) ) # printing the generator object's type  
for i in generator_object:  
    print( i )  

#~ <class 'generator'>
#~ Yield
#~ Keyword
#~ in
#~ Python
```

With yield, we can invoke functions rather than just returning values. Imagine, for example, that we create a function called square() that, when called, returns the square of a given number. The term return is used in this code. One more function, square(), uses the keyword yield to provide squares across a range of integers. In this case, we can combine the yield expression with the square() function to create a simple program. In the part below, look at the example code.

# Class Methods

[Reference Source](https://elfi-y.medium.com/python-properties-and-class-methods-a6c7ad69b0f1)

+ Python Class Attributes
+ Class and Static Methods
+ @property decorator

## Python Class Attributes

### Class and Static Attributes

```py
class MyClass:
  class_var = "class variable"
  def __init__(self, instance_var):
    self.instance_var = instance_var
def display(self):
     print("This is class variable :" + MyClass.class_var)
     print("This is instance variable :" + self.instance_var)
```

```shell
==================================>
>>> var = MyClass("instance variable")
>>> var.display()
This is class variable :class variable
This is instance variable :instance variable
```

class attribute — both exist on the class itself but not on the instance of class.

You can see class_var only exist on the class MyClass. This concept originates from C++ and C, which uses static keywords to make a variable a class variable.

### Class and Static Method

Compare them with the common instance methods:

```python
class MyClass:
    def instance_method(self):
        return 'instance method called', self
    @classmethod
    def classmethod(cls):
        return 'class method called', cls
    @staticmethod
    def staticmethod():
        return 'static method called'
```

+ **Instance Methods**: The first method `method` is an instance method. The parameter `self` points to an instance of MyClass. It can also access the class itself through `self.__class__` property.

```shell
>>> var = MyClass()
>>> var.instance_method()
(‘instance method called’, <__main__.MyClass object at 0x10cdd1e10>)
```

+ **Class Methods**: The `classmethod` with the `@classmethod` decorator is a class method. It takes a cls parameter that points to the class, and can only modify class states. It means the method operates on the class level, not the instance level. It can read or modify class variables that are shared across all instances, but it does not have access to instance-specific data unless explicitly passed an instance. This is useful for factory methods, which might create instance objects in diverse ways, or for modifying class-wide settings.

```shell
>>> var.classmethod()
(‘class method called’, <class ‘__main__.MyClass’>)
```

```python
class Car:
    total_cars = 0  # Class variable

    def __init__(self, model):
        self.model = model
        Car.total_cars += 1

    @classmethod
    def increase_total_cars(cls, amount):
        cls.total_cars += amount  # Modifying class state

    @classmethod
    def get_total_cars(cls):
        return cls.total_cars  # Accessing class state
# using increase_total_cars, can only modify the class state that reflects across all instances, demonstrating the ability to modify class-level data rather than instance-specific data.
```

+ **Static Methods: The `staticmethod` with `@staticmethod` decorator is a static method. It does not take the mandatory `self` nor a `cls` parameter. As a result, it can’t access class or instance state.

```shell
>>> var.staticmethod()
‘static method called’
```

Note that we can also call the later two methods on the class directly (As expected) but not on the instance method:

```shell
>> MyClass.instance_method()
Traceback (most recent call last):
 File “<stdin>”, line 1, in <module>
TypeError: instance_method() missing 1 required positional argument: ‘self’
>>> MyClass.classmethod()
(‘class method called’, <class ‘__main__.MyClass’>)
>>> MyClass.staticmethod()
‘static method called’
```

When choosing between `@staticmethod` and `@classmethod`, think about the latter requires access to the class object to call other class methods, while the former has no access needed to either class or instance objects. It can be moved up to the module-scope.

a `@staticmethod` in Python is similar to defining a normal function outside the class scope. It means the method does not access or modify the class state (cls) or instance state (self). It's used for functionality that is logically related to the class but doesn't need to interact with class-specific or instance-specific data.

On the use cases, one example with `@classmethod` is to have the flexibility to use it as **factory functions to initiate variants of class instances without using inheritance**. It is also a good practice to communicate your design intent with others:

```shell
>>> class MyCar:
...     def __init__(self, brand):
...         self.brand = brand
...     @classmethod
...     def benz(cls):
...         return cls(['benz'])
...     @classmethod
...     def bmw(cls):
...         return cls(['bmw'])
...     def __repr__(self):
...         return repr('This is my ' + self.brand )
```

Now to make car with certain brand.

```shell
>>> MyCar.benz()
MyCar(['benz'])
>>> MyCar.bmw()
MyCar(['bmw'])
```

`cls` refers to the class itself, `MyCar`, not a different class. When `@classmethod` is used, `cls` is automatically passed to the method and represents the class that the method is part of. So, when you call `MyCar.benz()` or `MyCar.bmw()`, `cls` is `MyCar`. This allows the method to instantiate new objects of `MyCar` using different predefined attributes. It's a way to create factory methods that can construct objects with specific configurations without needing separate subclasses for each configuration.

Using `car1 = MyCar.benz()` calls the class method `benz` of `MyCar`, which internally calls the class's constructor `__init__` with specific attributes (in this case, `['benz']` as the brand). This approach allows for creating instances with predefined configurations directly through class methods, without directly calling the constructor.

On the other hand, `car = MyCar()` attempts to create an instance of `MyCar` without any predefined configuration, and it will fail unless you modify the call to include the required `brand` parameter, like `MyCar('SomeBrand')`. This direct instantiation requires you to specify the attributes at the time of object creation, giving you more flexibility but less convenience for standard configurations.

Class methods like `benz` and `bmw` use `cls` to refer to the class itself, allowing them to work like factory methods that return instances of the class with specific predefined attributes.

Using `benz()` and `bmw()` class methods in `MyCar` is conceptually similar to calling `MyCar('benz')` and `MyCar('bmw')` directly. The class methods act as factory functions, creating instances of `MyCar` with specific attributes predefined (in this case, the brand). These methods provide a more readable and convenient way to create instances with common configurations, encapsulating the instantiation logic within the class itself, which can simplify code and improve maintainability.

# Property decorator

The @property decorator. Simply put, the `property()` method provides** an interface to instance attributes, aka, **`<strong class="mu fs">getter</strong>`** , **`<strong class="mu fs">setter</strong>`** and **`<strong class="mu fs">deleter</strong>`** .

It is common to allow dynamically setting and getting an instance attribute when you build a class, e.g. the brand attribute below:

```py
class Mycar:
 def __init__(self, brand=None):
   self._brand = brand
# getter
 def get_brand(self):
   print("Getting brand")
   return self._brand

# setter
  def set_brand(self, brand):
   self._brand = value
   print("Setting brand", self._brand)
```

```shell
benz = MyCar
benz.set_brand("benz")
benz.get_brand
===========================
Setting brand benz
Getting brand
```

This can do, but it's a bit tedious. Also what if the other developer comes in and wants to have another naming convention like instead of calling `set_brand` and `get_brand` , it will be `setting_brand` and `getting_brand` ?

use `property()` method in Python — a built-in function that creates and returns a `property` object, to solve this problem:

```
property(fget=None, fset=None, fdel=None, doc=None)
```

* `fget` is function to get value of the attribute
* `fset` is function to set value of the attribute
* `fdel` is function to delete the attribute
* `doc` is a docstring

So, it can also be changed to:

class Mycar:

```py
class Mycar:
 def __init__(self, brand=None):
   self._brand = brand
# getter
 def get_brand(self):
   print("Getting brand")
   return self._brand

# setter
  def set_brand(self, brand):
    self._brand = value
    print("Setting brand", self._brand)
  brand = property(get_brand, set_brand)

benz = MyCar
benz.brand = "benz" # NEW SYNTAX
benz.brand          # NEW SYNTAX
```

```shell
===========================
Setting brand benz
Getting brand
```

The `property` can be implemented as a decorator to your instance method:

```python
class Mycar:
 def __init__(self, brand=None):
   self._brand = brand
 @property
 def brand(self):
   print("Getting brand")
   return self._brand

 @brand.setter
 def brand(self, value):
   self._brand = value
   print("Setting brand", self._brand)

 @brand.deleter
 def brand(self):
   print("Deleting brand", self._brand)
   del self._brandbenz = MyCar
benz.brand = "benz" 
benz.brand        ===========================
Setting brand benz
Getting brand
```

And it can be generalised into this pattern:

```py
class C:
    def __init__(self):
        self._x = None
    @property
    def x(self):
        """I'm the 'x' property."""
        return self._x
    @x.setter
    def x(self, value):
        self._x = value
    @x.deleter
    def x(self):
        del self._x
```