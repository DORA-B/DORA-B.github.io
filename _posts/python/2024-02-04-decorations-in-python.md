---
title: Decoration In Python
date: 2024-02-04
categories: [Python]
tags: [python]     # TAG names should always be lowercase
---
Decorations in Python
=====================

References: [ref1](https://wiki.python.org/moin/PythonDecoratorLibrary), [ref2](https://www.datacamp.com/tutorial/decorators-python), [ref3](https://www.runoob.com/w3cnote/python-func-decorators.html)

A decorator is a design pattern in Python that allows a user to add new functionality to an existing object without modifying its structure.

Functions in Python are first-class citizens. This means that they support operations such as being passed as an argument, returned from a function, modified, and assigned to a variable. This property is crucial as it allows functions to be treated like any other object in Python.

## Assign function to variables

```py
def plus_one(number):
    return number + 1

add_one = plus_one
add_one(5)

# 6
```

## Defining Functions inside other functions

```py
def plus_one(number):
    def add_one(number):
        return number + 1


    result = add_one(number)
    return result
plus_one(4)

# 5
```

```py
def hi(name="yasoob"):
    print("now you are inside the hi() function")
 
    def greet():
        return "now you are in the greet() function"
 
    def welcome():
        return "now you are in the welcome() function"
 
    print(greet())
    print(welcome())
    print("now you are back in the hi() function")
 
hi()
#output:now you are inside the hi() function
#       now you are in the greet() function
#       now you are in the welcome() function
#       now you are back in the hi() function
```

## Passing Functions as Arguments to other Functions

```py
def plus_one(number):
    return number + 1

def function_call(function):
    number_to_add = 5
    return function(number_to_add)

function_call(plus_one)

# 6
```

## Functions Returning other Functions

```py
def hello_function():
    def say_hi():
        return "Hi"
    return say_hi
hello = hello_function()
hello()

# Hi
```

```py
def hi(name="yasoob"):
    def greet():
        return "now you are in the greet() function"
 
    def welcome():
        return "now you are in the welcome() function"
 
    if name == "yasoob":
        return greet
    else:
        return welcome
 
a = hi()
print(a)

#outputs: <function greet at 0x7f2143c01500>
```

## Nested Functions have access to the Enclosing Function's Variable Scope

```py
def print_message(message):
    "Enclosong Function"
    def message_sender():
        "Nested Function"
        print(message)

    message_sender()

print_message("Some random message")
# Some random message
```

```py
def uppercase_decorator(function):
    def wrapper():
        func = function()
        make_uppercase = func.upper()
        return make_uppercase

    return wrapper

def say_hi():
    return 'hello there'

decorate = uppercase_decorator(say_hi)
decorate()

# 'HELLO THERE'
```

However, Python provides a much easier way for us to apply decorators. We simply use the @ symbol before the function we'd like to decorate.

```py
@uppercase_decorator
def say_hi():
    return 'hello there'

say_hi()
# 'HELLO THERE'
# ==========================
# Compare to
# decorate = uppercase_decorator(say_hi)
# decorate()
# ==========================
```

## Applying Multiple Decorations to a Single Function

We can use multiple decorators to a single function. However, the decorators will be applied in the order that we've called them. Below we'll define another decorator that splits the sentence into a list. We'll then apply the `uppercase_decorator` and `split_string` decorator to a single function.

```py
import functools
def split_string(function):
    @functools.wraps(function)
    def wrapper():
        func = function()
        splitted_string = func.split()
        return splitted_string

    return wrapper 

@split_string
@uppercase_decorator
def say_hi():
    return 'hello there'
say_hi()

#  ['HELLO', 'THERE']
```

From the above output, we notice that the application of decorators is from the bottom up. Had we interchanged the order, we'd have seen an error since lists don't have an `upper` attribute. The sentence has first been converted to uppercase and then split into a list.

Python processes decorators from the innermost to the outermost.

```py
def decorator1(func):
    def wrapper(*args, **kwargs):
        print("Decorator 1: Before function call")
        result = func(*args, **kwargs)
        print("Decorator 1: After function call")
        return result
    return wrapper

def decorator2(func):
    def wrapper(*args, **kwargs):
        print("Decorator 2: Before function call")
        result = func(*args, **kwargs)
        print("Decorator 2: After function call")
        return result
    return wrapper

@decorator1
@decorator2
def target_function():
    print("Target function executing")

target_function()  
```

The output is

```shell
Decorator 1: Before function call
Decorator 2: Before function call
Target function executing
Decorator 2: After function call
Decorator 1: After function call  
```

<div class="admonition note">
<p class="first admonition-title">Note</p>
<p class="last">When stacking decorators, it's a common practice to use functools.wraps to ensure that the metadata of the original function is preserved throughout the stacking process. This helps maintain clarity and consistency in debugging and understanding the properties of the decorated function.</p>
</div>

## Accepting Arguments in Decorator Functions

```py
def decorator_with_arguments(function):
    def wrapper_accepting_arguments(arg1, arg2):
        print("My arguments are: {0}, {1}".format(arg1,arg2))
        function(arg1, arg2)
    return wrapper_accepting_arguments


@decorator_with_arguments
def cities(city_one, city_two):
    print("Cities I love are {0} and {1}".format(city_one, city_two))

cities("Nairobi", "Accra")

# My arguments are: Nairobi, Accra Cities I love are Nairobi and Accra
```

<div class="admonition note">
<p class="first admonition-title">Note</p>
<p class="last">It's essential to ensure that the number of arguments in the decorator (arg1, arg2 in this example) matches the number of arguments in the wrapped function (cities in this example). This alignment is crucial to avoid errors and ensure proper functionality when using decorators with arguments.
</p>
</div>

## Define General Purpose Decorators

```py
def a_decorator_passing_arbitrary_arguments(function_to_decorate):
    def a_wrapper_accepting_arbitrary_arguments(*args,**kwargs):
        print('The positional arguments are', args)
        print('The keyword arguments are', kwargs)
        function_to_decorate(*args)
    return a_wrapper_accepting_arbitrary_arguments

@a_decorator_passing_arbitrary_arguments
def function_with_no_argument():
    print("No arguments here.")

function_with_no_argument()
```

```shell
The positional arguments are ()
The keyword arguments are {}
No arguments here.
```

Using the decorator with positional arguments.

```py
@a_decorator_passing_arbitrary_arguments
def function_with_arguments(a, b, c):
    print(a, b, c)

function_with_arguments(1,2,3)
```

```shell
The positional arguments are (1, 2, 3)
The keyword arguments are {}
1 2 3
```

Keyword arguments are passed using keywords.

```py
Keyword arguments are passed using keywords.
```

```shell
The positional arguments are ()
The keyword arguments are {'first_name': 'Derrick', 'last_name': 'Mwiti'}
This has shown keyword arguments
```

<div class="admonition note">
<p class="first admonition-title">Note</p>
<p class="last">The use of **kwargs in the decorator allows it to handle keyword arguments. This makes the general-purpose decorator versatile and capable of handling a variety of argument types during function calls.
</p>
</div>

## Passing Arguments to the Decorator

```py
def decorator_maker_with_arguments(decorator_arg1, decorator_arg2, decorator_arg3):
    def decorator(func):
        def wrapper(function_arg1, function_arg2, function_arg3) :
            "This is the wrapper function"
            print("The wrapper can access all the variables\n"
                  "\t- from the decorator maker: {0} {1} {2}\n"
                  "\t- from the function call: {3} {4} {5}\n"
                  "and pass them to the decorated function"
                  .format(decorator_arg1, decorator_arg2,decorator_arg3,
                          function_arg1, function_arg2,function_arg3))
            return func(function_arg1, function_arg2,function_arg3)

        return wrapper

    return decorator

pandas = "Pandas"
@decorator_maker_with_arguments(pandas, "Numpy","Scikit-learn")
def decorated_function_with_arguments(function_arg1, function_arg2,function_arg3):
    print("This is the decorated function and it only knows about its arguments: {0}"
           " {1}" " {2}".format(function_arg1, function_arg2,function_arg3))

decorated_function_with_arguments(pandas, "Science", "Tools")
```

```shell
The wrapper can access all the variables
    - from the decorator maker: Pandas Numpy Scikit-learn
    - from the function call: Pandas Science Tools
and pass them to the decorated function
This is the decorated function, and it only knows about its arguments: Pandas Science Tools

```

## Debugging Decorators

Decorators wrap functions. The original function name, its docstring, and parameter list are all hidden by the wrapper closure: For example, when we try to access the decorated_function_with_arguments metadata, we'll see the wrapper closure's metadata. A challenge arises when debugging python, to solve it, Python provides `functools.wraps` [decorator](https://docs.python.org/3/library/functools.html#functools.wraps). This decorator copies the lost metadata from the undecorated function to the decorated closure.

```py
import functools

def uppercase_decorator(func):
    @functools.wraps(func)
    def wrapper():
        return func().upper()
    return wrapper
@uppercase_decorator
def say_hi():
    "This will say hi"
    return 'hello there'

say_hi()

# 'HELLO THERE'
```

When we check the `say_hi` metadata, we notice that it is now referring to the function's metadata and not the wrapper's metadata.

```py
say_hi.__name__
# 'say_hi'
say_hi.__doc__
# 'This will say hi'
```

<div class="admonition note">
<p class="first admonition-title">Note</p>
<p class="last">It is advisable and good practice to always use `functools.wraps` when defining decorators. It will save you a lot of headaches in debugging
</p>
</div>


## Insert logger into Function

Create a wrapper function, which can enable use to output a log file with specific name

```py
from functools import wraps
 
def logit(logfile='out.log'):
    def logging_decorator(func):
        @wraps(func)
        def wrapped_function(*args, **kwargs):
            log_string = func.__name__ + " was called"
            print(log_string)
            # 打开logfile，并写入内容
            with open(logfile, 'a') as opened_file:
                # 现在将日志打到指定的logfile
                opened_file.write(log_string + '\n')
            return func(*args, **kwargs)
        return wrapped_function
    return logging_decorator
 
@logit()
def myfunc1():
    pass
 
myfunc1()
# Output: myfunc1 was called
# 现在一个叫做 out.log 的文件出现了，里面的内容就是上面的字符串
 
@logit(logfile='func2.log')
def myfunc2():
    pass
 
myfunc2()
# Output: myfunc2 was called
# 现在一个叫做 func2.log 的文件出现了，里面的内容就是上面的字符串
```

## Decorator Class

Sometimes there will be more complex work for a wrapper function to do, so we need a wrapper class

```py
from functools import wraps
 
class logit(object):
    def __init__(self, logfile='out.log'):
        self.logfile = logfile
 
    def __call__(self, func):
        @wraps(func)
        def wrapped_function(*args, **kwargs):
            log_string = func.__name__ + " was called"
            print(log_string)
            # 打开logfile并写入
            with open(self.logfile, 'a') as opened_file:
                # 现在将日志打到指定的文件
                opened_file.write(log_string + '\n')
            # 现在，发送一个通知
            self.notify()
            return func(*args, **kwargs)
        return wrapped_function
 
    def notify(self):
        # logit只打日志，不做别的
        pass
```

This implementation has a better additional advantage. It's cleaner than nested functions, and wrapping a function still uses the same syntax as before:

```py
@logit()
def myfunc1():
    pass
```

If we want a more complex function that will be integrated into this logit class, which is the email function.

```py
class email_logit(logit):
    '''
    一个logit的实现版本，可以在函数调用时发送email给管理员
    '''
    def __init__(self, email='admin@myproject.com', *args, **kwargs):
        self.email = email
        super(email_logit, self).__init__(*args, **kwargs)
 
    def notify(self):
        # 发送一封email到self.email
        # 这里就不做实现了
        pass
```

From now on, `@email_logit` will have the same effect as `@logit`, but in addition to logging, an additional email will be sent to the administrator.


# Several interesting examples
Examples are from Indently YT channels [github repo](https://github.com/indently/five_decorators/tree/main/decorators)

## Retry: Functools.wraps
```py
import time
from functools import wraps
from typing import Callable, Any
from time import sleep


def retry(retries: int = 3, delay: float = 1) -> Callable:
    """
    Attempt to call a function, if it fails, try again with a specified delay.

    :param retries: The max amount of retries you want for the function call
    :param delay: The delay (in seconds) between each function retry
    :return:
    """

    # Don't let the user use this decorator if they are high
    if retries < 1 or delay <= 0:
        raise ValueError('Are you high, mate?')

    def decorator(func: Callable) -> Callable:
        @wraps(func)
        def wrapper(*args, **kwargs) -> Any:
            for i in range(1, retries + 1):  # 1 to retries + 1 since upper bound is exclusive

                try:
                    print(f'Running ({i}): {func.__name__}()')
                    return func(*args, **kwargs)
                except Exception as e:
                    # Break out of the loop if the max amount of retries is exceeded
                    if i == retries:
                        print(f'Error: {repr(e)}.')
                        print(f'"{func.__name__}()" failed after {retries} retries.')
                        break
                    else:
                        print(f'Error: {repr(e)} -> Retrying...')
                        sleep(delay)  # Add a delay before running the next iteration

        return wrapper

    return decorator


@retry(retries=3, delay=1)
def connect() -> None:
    time.sleep(1)
    raise Exception('Could not connect to internet...')


def main() -> None:
    connect()
```

## Cache: functools.cache

```py
import time
from functools import cache


@cache
def count_vowels(text: str) -> int:
    """
    A function that counts all the vowels in a given string.

    :param text: The string to analyse
    :return: The amount of vowels as an integer
    """
    vowel_count: int = 0

    # Pretend it's an expensive operation
    print(f'Bot: Counting vowels in: "{text}"...')
    time.sleep(2)

    # Count those damn vowels
    for letter in text:
        if letter in 'aeiouAEIOU':
            vowel_count += 1

    return vowel_count


def main() -> None:
    while True:
        user_input: str = input('You: ')

        if user_input == '!info':
            print(f'Bot: {count_vowels.cache_info()}')
        elif user_input == '!clear':
            print('Bot: Cache cleared!')
            count_vowels.cache_clear()
        else:
            print(f'Bot: "{user_input}" contains {count_vowels(user_input)} vowels.')
```


## A timing method to some execution in the code

```py
import time
from time import perf_counter, sleep
from functools import wraps
from typing import Callable, Any


def get_time(func: Callable) -> Callable:
    @wraps(func)
    def wrapper(*args, **kwargs) -> Any:

        # Note that timing your code once isn't the most reliable option
        # for timing your code. Look into the timeit module for more accurate
        # timing.
        start_time: float = perf_counter()
        result: Any = func(*args, **kwargs)
        end_time: float = perf_counter()

        print(f'"{func.__name__}()" took {end_time - start_time:.3f} seconds to execute')
        return result

    return wrapper


# Sample function 1
@get_time
def connect() -> None:
    print('Connecting...')
    sleep(2)
    print('Connected!')


# Sample function 2
@get_time
def fifty_million_loops() -> None:
    fifty_million: int = int(5e7)

    print('Looping...')
    for _ in range(fifty_million):
        pass

    print('Done looping!')


def main() -> None:
    fifty_million_loops()
    connect()
```


## An Exit after execution

```py
import atexit
@atexit.register
def exit_handler() -> None:
    print("We're exiting now!")


def main() -> None:
    for i in range(10):
        print(2**i)


if __name__ == "__main__":
    main()
    atexit.unregister(exit_handler)
    1 / 0
```

- Another example

```py
import atexit
import sqlite3

cxn = sqlite3.connect("db.sqlite3")


def init_db():
    cxn.execute("CREATE TABLE IF NOT EXISTS memes (id INTEGER PRIMARY KEY, meme TEXT)")
    print("Database initialised!")


@atexit.register
def exit_handler():
    cxn.commit()
    cxn.close()
    print("Closed database!")


if __name__ == "__main__":
    init_db()
    1 / 0
    ...
```