---
title: lambda function in C++
date: 2023-04-20
categories: [DDS]
tags: [dds, ros2]     # TAG names should always be lowercase
---
# Using IDL to define a data type

Supported IDL types
[https://fast-dds.docs.eprosima.com/en/latest/fastddsgen/dataTypes/dataTypes.html]
+ Primitive types

|IDL|c++11|
|:---|:---|
|char|char|
|octet|uint8_t|
|short	|int16_t|
|unsigned short|uint16_t|
|long	|int32_t|
|unsigned long	|uint32_t|
|long long	|int64_t|
|unsigned long long	|uint64_t|
|float	|float|
|double|	double|
|long double	|long double|
|boolean	|bool|
|string	|std::string|

+ Arrays

|IDL|c++11|
|:---|:---|
|char a[5]|	std::array<char,5> a |
|octet a[5] |	std::array<uint8_t,5> a|
|short a[5]	|std::array<int16_t,5> a|
|unsigned short a[5]	|std::array<uint16_t,5> a|
|long a[5]	|std::array<int32_t,5> a|
|unsigned long a[5]	|std::array<uint32_t,5> a|
|long long a[5]|	std::array<int64_t,5> a|
|unsigned long long a[5]	|std::array<uint64_t,5> a|
|float a[5]	|std::array<float,5> a|
|double a[5]	|std::array<double,5> a|

+ Sequences

|IDL|c++11|
|:---|:---|
|sequence<char>	|std::vector<char>|
|sequence<octet>	|std::vector<uint8_t>|
|sequence<short>|	std::vector<int16_t>|
|sequence<unsigned short>	|std::vector<uint16_t>|
|sequence<long>	|std::vector<int32_t>|
|sequence<unsigned long>	|std::vector<uint32_t>|
|sequence<long long>	|std::vector<int64_t>|
|sequence<unsigned long long>	|std::vector<uint64_t>|
|sequence<float>	| std::vector<float>|
|sequence<double>	| std::vector<double>|


+ Maps

|IDL|c++11|
|:---|:---|
|map<char, unsigned long long> | std::map<char, uint64_T>|
+ Structures
The following IDL structure:
```cpp
  struct Structure
  {
      octet octet_value;
      long long_value;
      string string_value;
  };
```
Be converted to
```cpp
class Structure
{
public:
    Structure();
    ~Structure();
    Structure(const Structure &x);
    Structure(Structure &&x);
    Structure& operator=(const Structure &x);
    Structure& operator=(Structure &&x);

    void octet_value(uint8_t _octet_value);
    uint8_t octet_value() const;
    uint8_t& octet_value();
    void long_value(int64_t _long_value);
    int64_t long_value() const;
    int64_t& long_value();
    void string_value(const std::string
        &_string_value);
    void string_value(std::string &&_string_value);
    const std::string& string_value() const;
    std::string& string_value();

private:
    uint8_t m_octet_value;
    int64_t m_long_value;
    std::string m_string_value;
};
```
Inherit from other structures
```cpp
struct ParentStruct
{
    octet parent_member;
};

struct ChildStruct : ParentStruct
{
    long child_member;
};

//convert to 

class ParentStruct
{
    octet parent_member;
};

class ChildStruct : public ParentStruct
{
    long child_member;
};
```
+ Unions
```cpp
union Union switch(long)
{
   case 1:
    octet octet_value;
  case 2:
    long long_value;
  case 3:
    string string_value;
};
// convert to 
class Union
{
public:
    Union();
    ~Union();
    Union(const Union &x);
    Union(Union &&x);
    Union& operator=(const Union &x);
    Union& operator=(Union &&x);

    void d(int32_t __d);
    int32_t _d() const;
    int32_t& _d();

    void octet_value(uint8_t _octet_value);
    uint8_t octet_value() const;
    uint8_t& octet_value();
    void long_value(int64_t _long_value);
    int64_t long_value() const;
    int64_t& long_value();
    void string_value(const std::string
        &_string_value);
    void string_value(std:: string &&_string_value);
    const std::string& string_value() const;
    std::string& string_value();

private:
    int32_t m__d;
    uint8_t m_octet_value;
    int64_t m_long_value;
    std::string m_string_value;
};
```
+ Bitsets
Bitsets are a special kind of sturcture, which encloses a set of bits. A bitset can represent up to 64 bits. Each member is defined as bitfield and eases the access to a part of the bitset.
Blow is a example named MyBitset, which store a total of 25 bits (3+10+12) and will require 32 bits in memory.
(Which represents a rule that lowest primitive to store the bitset's size)
+ The bitfield `a` allows us to access to the first 3 bits (0..2).
+ The bitfield `b` allows us to access to the next 10 bits (3..12).
+ The bitfield `c` allows us to access to the next 12 bits (13..24).
```cpp
bitset MyBitset
{
    bitfield<3> a;
    bitfield<10> b;
    bitfield<12, int> c;
};
// resulting C++ code will be similar to 
class MyBitset
{
public:
    void a(char _a);
    char a() const;

    void b(uint16_t _b);
    uint16_t b() const;

    void c(int32_t _c);
    int32_t c() const;

private:
    std::bitset<25> m_bitset;
};
```
Internally, it is stored as a std::bitset. For each bitfield, get() and set() member functions are generated with the smaller possible primitive unsigned type to access it. `In the case of bitfield ¡®c¡¯, the user has established that this accessing type will be int, so the generated code uses int32_t instead of automatically use uint16_t`.
What is more, Bitsets can inherit from other bitsets, extending their member set.
```cpp
bitset ParentBitset
{
  bitfield<3> parent_member;
};

bitset ChildBitset : ParentBitset
{
  bitfield<10> child_member;
};
// result converting is
class ParentBitset
{
    std::bitset<3> parent_member;
};

class ChildBitset : public ParentBitset
{
    std::bitset<10> child_member;
};
```

+ Enumerations
```cpp
enum Enumeration
{
    RED,
    GREEN,
    BLUE
};
// convert result is
enum Enumeration : uint32_t
{
    RED,
    GREEN,
    BLUE
};
```
+ Bitmasks
Bitmasks are a special kind of Enumeration to manage masks of bits, which allows defining bit masks based on their position.
```cpp
@bit_bound(8)
bitmask MyBitMask
{
    @position(0) flag0,
    @position(1) flag1,
    @position(4) flag4,
    @position(6) flag6,
    flag7
};
// convert result is 
enum MyBitMask : uint8_t
{
    flag0 = 0x01 << 0,
    flag1 = 0x01 << 1,
    flag4 = 0x01 << 4,
    flag6 = 0x01 << 6,
    flag7 = 0x01 << 7
};
```
+ Data types with a key
In order to use keyed topics, the user should defind some key members inside the sturcture. 
Using `@Key` annotation before the members of the structure that are used as keys.
These tags will be automatically detected by DDS-GEN. DDS-GEN will generates the serialization methods for they key generation function in TopicDataType(getKey(),`this function will obtain the 128-bit MD5 digest of the big-dndian serialization of the Key Members`).

```cpp
struct MyType
{
    @Key long id;
    @Key string type;
    long positionX;
    long positionY;
};
```
# Including other IDL files

Fast DDS-Gen uses a C/C++ preprocessor for this purpose, and #include directive can be used to include an IDL file.
```cpp
#include "OtherFile.idl"
#include <AnotherFile.idl>
```
If Fast DDS-Gen does not find a C/C++ preprocessor in default system paths, the preprocessor path can be specified using parameter -ppPath. The parameter -ppDisable can be used to disable the usage of the C/C++ preprocessor.

# Annotations

Allow users to define and use their own  annotations as defined in the OMG in the `OMG IDL 4.2 specification`.
User annotations will be passed to TypeObject generated code if the -typeobject argument was used.

```
@annotation MyAnnotation
{
    long value;
    string name;
};
```
buildin annotations:
[https://fast-dds.docs.eprosima.com/en/latest/fastddsgen/dataTypes/dataTypes.html#annotations]

# Forward Declaration

Fast DDS-Gen supports forward declarations. This allows declaring inter-dependant structures, unions, etc.

```cpp
struct ForwardStruct;
union ForwardUnion;
struct ForwardStruct
{
    ForwardUnion fw_union;
};
union ForwardUnion switch (long)
{
    case 0:
        ForwardStruct fw_struct;
    default:
        string empty;
};
```

# IDL 4.2 comments

+ // [comments]
+ /* [comments] */