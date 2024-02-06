This documents mainly discuss the transfer from Value Categories to Std::Move 

==[reference of value categories](https://en.cppreference.com/w/cpp/language/value_category)==
==[reference of std::move](https://en.cppreference.com/w/cpp/utility/move)==

# Value Categories

Each C++ expression is characterized by two independent properties: a type and a value category. 
Each expression has some non-reference type, and each expression belongs to exactly one of the three primary value categories: prvalue, xvalue, and lvalue.

+ glvalue: (“generalized” lvalue) is an expression whose evaluation determines the identity of an object or function;
A glvalue expression is either lvalue or xvalue.
  Properties:

  + A glvalue may be implicitly converted to a prvalue with lvalue-to-rvalue, array-to-pointer, or function-to-pointer implicit conversion.
  + A glvalue may be polymorphic: the dynamic type of the object it identifies is not necessarily the static type of the expression.
  A glvalue can have incomplete type, where permitted by the expression.


+ prvalue: (“pure” rvalue) is an expression whose evaluation
  + computes the value of an operand of a built-in operator (such prvalue has no result object), or
  + initializes an object (such prvalue is said to have a result object).
  The result object may be a variable, an object created by new-expression, a temporary created by temporary materialization, or a member thereof. Note that non-void discarded expressions have a result object (the materialized temporary). Also, every class and array prvalue has a result object except when it is the operand of decltype;
  * Same as rvalue.
  * A prvalue cannot be polymorphic: the dynamic type of the object it denotes is always the type of the expression.
  * A non-class non-array prvalue cannot be cv-qualified, unless it is materialized in order to be bound to a reference to a cv-qualified type (since C++17). (Note: a function call or cast expression may result in a prvalue of non-class cv-qualified type, but the cv-qualifier is generally immediately stripped out.)
  * A prvalue cannot have incomplete type (except for type void, see below, or when used in decltype specifier).
  * A prvalue cannot have abstract class type or an array thereof.

+ xvalue: (an “eXpiring” value) is a glvalue that denotes an object whose resources can be reused;
  The following expressions are xvalue expressions:
  a.m, the member of object expression, where a is an rvalue and m is a non-static data member of an object type;
  a.*mp, the pointer to member of object expression, where a is an rvalue and mp is a pointer to data member;
  a ? b : c, the ternary conditional expression for certain b and c (see definition for detail);
  a function call or an overloaded operator expression, whose return type is rvalue reference to object, such as std::move(x);
  a[n], the built-in subscript expression, where one operand is an array rvalue;
  a cast expression to rvalue reference to object type, such as static_cast<char&&>(x); (since C++11)
  any expression that designates a temporary object, after temporary materialization. (since C++17)
  a move-eligible expression. (since C++23)
  Same as rvalue (below).
  Same as glvalue (below).
  In particular, like all rvalues, xvalues bind to rvalue references, and like all glvalues, xvalues may be polymorphic, and non-class xvalues may be cv-qualified.
  + Same as rvalue.
  + Same as glvalue.
In particular, like all rvalues, xvalues bind to rvalue references, and like all glvalues, xvalues may be polymorphic, and non-class xvalues may be cv-qualified.


+ lvalue: (so-called, historically, because lvalues could appear on the left-hand side of an assignment expression) is a glvalue that is not an xvalue;
  + Same as glvalue.
  + Address of an lvalue may be taken by built-in address-of operator: &++i and &std::endl are valid expressions.
  + A modifiable lvalue may be used as the left-hand operand of the built-in assignment and compound assignment operators.
  + An lvalue may be used to initialize an lvalue reference; this associates a new name with the object identified by the expression.

+ rvalue: (so-called, historically, because rvalues could appear on the right-hand side of an assignment expression) is a prvalue or an xvalue.

  Properties:

  + Address of an rvalue cannot be taken by built-in address-of operator: &int(), &i++, &42, and &std::move(x) are invalid.
  + An rvalue can't be used as the left-hand operand of the built-in assignment or compound assignment operators.
  + An rvalue may be used to initialize a const lvalue reference, in which case the lifetime of the object identified by the rvalue is extended until the scope of the reference ends.
  + An rvalue may be used to initialize an rvalue reference, in which case the lifetime of the object identified by the rvalue is extended until the scope of the reference ends.
  + When used as a function argument and when two overloads of the function are available, one taking rvalue reference parameter and the other taking lvalue reference to const parameter, an rvalue binds to the rvalue reference overload (thus, if both copy and move constructors are available, an rvalue argument invokes the move constructor, and likewise with copy and move assignment operators). (since C++ 11)


# std::move

std::move is used to indicate that an object `t` may be "moved from", i.e. allowing the efficient transfer of resources from `t` to another object.

In particular, std::move produces an xvalue expression that identifies its argument `t`. It is exactly equivalent to a static_cast to an rvalue reference type.

std::move is need for converting an lvalue reference into an rvalue reference, which allows the compiler to move the resources from one object to another instead of copying them. This can improve the performance and efficiency of the code, especially when dealing with large or complex objects.

Some examples of when std::move can be useful are:

In move constructors and move assignment operators, to transfer the ownership of the resources from the source object to the target object. For example:

```cpp
ArrayWrapper ( ArrayWrapper && other) : _p_vals ( other._p_vals ) , _metadata ( std::move ( other._metadata ) ) { other._p_vals = NULL; }
```

In swap functions, to exchange the values of two objects with less copying. For example:

```cpp
template <class T> swap (T& a, T& b) {  
T tmp (std::move(a)); // we've moved a to tmp  
a = std::move(b); // we've moved b to a  
b = std::move(tmp); // we've moved tmp to b 
}
In return statements, to avoid unnecessary copies of temporary objects. For example:
T foo() {  
T t;  // do something with t  
return std::move(t); // move t to the caller 
}
```

However, std::move should be used only when necessary, as it can also have some drawbacks, such as:

+ Preventing compiler optimizations, such as copy elision or return value optimization, which can eliminate copies without std::move.
+ Leaving the source object in an unspecified state, which can cause bugs or undefined behavior if it is used later.
+ Breaking class invariants or expectations, such as constness or immutability, if the moved object is modified by the target object.

```cpp
// t is the object to be moved
static_cast<typename std::remove_reference<T>::type&&>(t)
```

```cpp
std::vector<std::string> v;
std::string str = "example";
v.push_back(std::move(str)); // str is now valid but unspecified
str.back(); // undefined behavior if size() == 0: back() has a precondition !empty()
if (!str.empty())
    str.back(); // OK, empty() has no precondition and back() precondition is met
 
str.clear(); // OK, clear() has no preconditions
```

Also, the standard library functions called with xvalue arguments may assume the argument is the only reference to the object; if it was constructed from an lvalue with std::move, no aliasing checks are made. However, self-move-assignment of standard library types is guaranteed to place the object in a valid (but usually unspecified) state:

```cpp
std::vector<int> v = {2, 3, 3};
v = std::move(v); // the value of v is unspecified
```
