---
title: lambda function in C++
date: 2023-10-02
categories: [C++]
tags: [c++]     # TAG names should always be lowercase
---
A lambda expression in C++ is a way of defining an anonymous function object (also called a closure) that can be used inline or passed as an argument to another function. It was introduced in C++11 as a more convenient and concise way of creating functors, which are objects that can be called like functions.

A lambda expression has the following general syntax:

```cpp
[ captures ] ( params ) specs -> ret { body }
```

where:

captures is a list of variables from the enclosing scope that are used in the lambda body. It can specify how the variables are captured: by value or by reference. It can also be empty, meaning no variables are captured.
params is a list of parameters for the lambda function, similar to a normal function parameter list, but without default values or variadic arguments. It can also be omitted if the lambda takes no arguments.
specs is a list of specifiers for the lambda function, such as mutable, constexpr, noexcept, etc. It can also be omitted if no specifiers are needed.
ret is the return type of the lambda function. It can also be omitted if the return type can be deduced from the body or if the lambda returns nothing.
body is the code block that defines the behavior of the lambda function.
For example, this lambda expression defines a function that takes two integers and returns their sum:

```cpp
[](int x, int y) -> int { return x + y; }
```

This lambda expression defines a function that takes no arguments and prints "Hello, world" to the standard output:

```cpp
{ std::cout << "Hello, world" << std::endl; }
```

Lambda expressions are often used with standard library algorithms or asynchronous functions that take functions as arguments. For example, this code uses a lambda expression to count how many elements in a vector are greater than 5:

```cpp
std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8, 9}; 
int count = std::count_if(v.begin(), v.end(), [](int n) { return n > 5; });
```

For more information and examples of lambda expressions in C++, you can refer to these web search results: