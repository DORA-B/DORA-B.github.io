---
title: Lambda Function, Callable and Optional in Python
date: 2024-03-10
categories: [Python]
tags: [python]     # TAG names should always be lowercase
---
Lambda Function, Callable and Optional in Python
================================================

Here are several code snippets:

```python
# return a function that be used to dynamic replacements based on the contents of the replacer
def repl_dict2func(replacer: Dict[str, str]) -> Callable[[Match], str]:
    return lambda a: replacer.get(a.group(), a.group())
  
# where call this function
_str = str(inst.base_inst)
_str = reg_finder.sub(repl_dict2func(reg_repl), _str)
```

    In Python, a`Callable` is a type hint that indicates an object is "callable," meaning it can be called like a function. This includes functions, methods, classes, and any objects that implement the special `__call__` method. When you see `Callable` in a type hint, it often includes specifications about the arguments it expects and the type of value it returns.

The function `repl_dict2func` you've shared demonstrates a common use of `Callable`. It takes a dictionary mapping strings to strings (`replacer`) and returns a new function. This returned function is designed to be used with methods like `re.sub()` in Python's `re` module, which can accept a function as a replacement argument to dynamically determine the replacement string based on matches.

**Function Definition** : `def repl_dict2func(replacer: Dict[str, str]) -> Callable[[Match], str]:`

* It defines a function `repl_dict2func` that takes a single argument `replacer`, which is a dictionary mapping strings to strings.
* It returns a `Callable`. The `Callable[[Match], str]` type hint indicates that the returned function will take a single argument of type `Match` (a match object from the `re` module) and will return a string.

**Lambda Function** : `return lambda a: replacer.get(a.group(), a.group())`

* This line returns an anonymous function (lambda) that takes a single argument `a`.
* `a.group()` gets the matched string from the match object.
* `replacer.get(a.group(), a.group())` tries to find the matched string in the `replacer` dictionary. If the match is found in the dictionary, it returns the corresponding value. If not, it returns the matched string itself.
* Essentially, this lambda function looks up each match in the `replacer` dictionary to see if it should be replaced with something else. If a replacement is defined, it uses that; otherwise, it just returns the match as is.

This `repl_dict2func` function is useful for creating dynamic replacement functions for regular expression substitutions, allowing the replacement logic to be determined by the contents of a dictionary.

### Lambda Function

+ **Basic Lambda Function**, This lambda function takes two arguments x and y and returns their sum. It's equivalent to the following regular function:

  ```python
  lambda arguments: expression
  add = lambda x, y: x + y
  print(add(5, 3))  # Output: 8
  ```
+ **Lambda with No Arguments**

  ```python
  get_five = lambda: 5
  print(get_five())  # Output: 5
  ```
+ **Lambda in a Higher-Order Function:** In this example, `map` is a higher-order function that applies the lambda function to each item in the `numbers` list.

  ```python
  numbers = [1, 2, 3, 4, 5]
  squared = map(lambda x: x**2, numbers)
  print(list(squared))  # Output: [1, 4, 9, 16, 25]
  ```
+ **Lambda with Conditional Expressions**

  ```python
  max = lambda a, b: a if a > b else b
  print(max(5, 10))  # Output: 10
  ```
+ **Return Lambda function:** call a returned lambda function

    ```python
    def myfunc(n):
    return lambda a : a * n

    mydoubler = myfunc(2)

    print(mydoubler(11))
    ```

+ Lambda Function with `get` function

  ```python
          extract_method = {
              'guest_arch': self.extract_state,
          }.get(isa, lambda state: dprint_exit('Unsupported guest: ' + isa))  
  ```

    A dictionary named `extract_method` is used to map guest instruction set architectures (ISAs) to specific functions that extract register state information. The `get` method is being used on this dictionary with two arguments:

    1. `isa`: This is the key that the `get` method tries to find in the `extract_method` dictionary. It represents the ISA of the guest, such as 'guest_arch'.
    2. `lambda state: dprint_exit('Unsupported guest: ' + isa)`: This is a lambda function that acts as a default value in case `isa` is not found in the `extract_method` dictionary. If the ISA is not supported (i.e., not present in the dictionary), this lambda function will be returned and called, executing `dprint_exit` with a message that the guest ISA is unsupported.

    Here's a breakdown of how it works:

    * If `isa` is found in the `extract_method` dictionary (for example, if `isa` is 'guest_arch'), the corresponding function (`self.extract_state` in this case) is retrieved and stored in the `extract_method` variable.
    * If `isa` is not found in the dictionary, the lambda function is returned and stored in the `extract_method` variable. When this lambda function is called with a state, it will execute `dprint_exit` to print or log an error message indicating that the guest ISA is not supported.

    This approach provides a clean and concise way to handle different cases based on the ISA and to provide a clear error message if the ISA is not recognized.

### **Optional**

```python
def hex_str_to_bin(token: str, const_bit: int = 32) -> Optional[List[int]]:
    if hex_matcher.match(token):
        num = int(token, 16)
        return gen_bin_from_int(num, const_bit)
    else:
        return None
```

In Python, `Optional` is a type hint from the `typing` module that indicates a variable can either have a specific type or be `None`. When you use `Optional[SomeType]`, it's equivalent to saying that the variable could be either of type `SomeType` or it could be `None`.

In the function `hex_str_to_bin` you've shared:

* `token: str` indicates that the `token` argument should be of type `str` (string).
* `const_bit: int = 32` indicates that `const_bit` is an optional parameter with a default value of 32, and it should be of type `int` (integer).
* `-> Optional[List[int]]` indicates that the function returns either `None` or a list of integers (`List[int]`).

The function checks if the provided `token` matches a hexadecimal number pattern (presumably, `hex_matcher` is a compiled regular expression for matching hexadecimal numbers). If it does, the function converts the hexadecimal string to an integer, then calls `gen_bin_from_int` to generate a list of integers (presumably binary representation) and returns it. If `token` doesn't match the expected pattern, the function returns `None`.

So, the use of `Optional` here signifies that the return value of `hex_str_to_bin` can be either a list of integers (when the conversion is successful) or `None` (when the input doesn't match a hexadecimal pattern).
