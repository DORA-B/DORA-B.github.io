---
title: Some python tips from cs50 and 39 keyworkds 
date: 2025-03-09
categories: [Python]
tags: [python]     # TAG names should always be lowercase
---

# Tuple
> A tuple is a collection which is ordered and unchangeable.

## Returning Multiple Values Using Tuples
A function can return multiple values as a tuple, which is an immutable (unchangeable) sequence type in Python. Here’s an example function that returns a tuple:
```py
def func():
    a = 1
    b = 2
    c = 3
    d = 4
    return a, b, c, d
    # Python automatically packs the returned values into a tuple. 
# unpacking You can then unpack these values into separate variables:

(reta, retb, retc, retd) = func()


# Once you have unpacked the values from the tuple into individual variables (reta, retb, retc, retd), these variables are independent of the tuple and behave like normal Python variables. You can modify them as you wish:

reta += 1

# The immutability of tuples only means that you cannot change the contents of the tuple itself after it has been created. For example, if you had kept the tuple intact:
results = func()

# NOTE results[0] = 10  # This would raise an error because you can't change elements of a tuple

# In this case, trying to modify an element of the tuple directly will raise a TypeError because tuples do not support item assignment.
```

# Class property method and getter, setter and its validation

# operator overloading

- [Special method names](https://docs.python.org/3/reference/datamodel.html)

# unpacking
- *args, **kwargs


## filter 

```py
# Function to check if a number is even
def even(n):
    return n % 2 == 0
a = [1, 2, 3, 4, 5, 6]
b = filter(even, a)
# Convert filter object to a list
print(list(b))  
# [2, 4, 6] 
```
- Using filter() with lambda
```py
a = [1, 2, 3, 4, 5, 6]
b = filter(lambda x: x % 2 == 0, a)
print(list(b))  
# [2, 4, 6]
```
- Combining filter() with Other Functions

```python
a = [1, 2, 3, 4, 5, 6]

# First, filter even numbers
b = filter(lambda x: x % 2 == 0, a)

# Then, double the filtered numbers
c = map(lambda x: x * 2, b)
print(list(c))  
# [4, 8, 12]
```


## `__format__` method in a class
- can return different format according to a specific format 
- 
