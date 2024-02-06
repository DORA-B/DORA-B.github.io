1. Understand what smart pointers are and how they differ from raw pointers. Smart pointers are classes that manage the lifetime of dynamically allocated objects using reference counting or ownership semantics. Raw pointers are just plain addresses that do not have any responsibility for the objects they point to.

2. Learn how to create and use std::shared_ptr, which is a smart pointer that retains shared ownership of an object through a pointer. Several std::shared_ptr objects may own the same object, and the object is destroyed when the last remaining std::shared_ptr owning it is destroyed. You can use the constructor, the assignment operator, or the reset() method to associate a std::shared_ptr with an object. You can also use the get() method to access the raw pointer, or the use_count() method to get the number of std::shared_ptr objects sharing ownership of the object.
3. Learn how to use std::make_shared, which is a function template that constructs an object of type T and wraps it in a std::shared_ptr using args as the parameter list for the constructor of T. This is more efficient than creating a std::shared_ptr with a raw pointer, because it allocates only one memory block for both the object and the control data required for reference counting. You can also use std::make_shared to create arrays of objects since C++20.
Read some tutorials and examples on how to use std::shared_ptr and std::make_shared.

Study this_thread::sleep_for and chrono:

1. Understand what threads are and how they can be created and managed in C++. Threads are units of execution that can run concurrently within a process, sharing its resources. You can use the std::thread class to create and join threads, or the std::async function to create asynchronous tasks that return futures.
2. Learn how to use this_thread::sleep_for, which is a function template that blocks the execution of the current thread for at least the specified sleep_duration. This function may block for longer than sleep_duration due to scheduling or resource contention delays. The sleep_duration is a std::chrono::duration object that represents a time interval.
3. Learn how to use chrono, which is a header file that provides utilities for time measurement and manipulation. You can use various clocks, such as system_clock, steady_clock, or high_resolution_clock, to get the current time point or duration. You can also use various duration types, such as nanoseconds, microseconds, milliseconds, seconds, minutes, or hours, to represent time intervals with different precision and units. You can also use literals, such as 10s or 500ms, to create duration objects with user-defined suffixes.
Read some tutorials and examples on how to use this_thread::sleep_for and chrono.

https://en.cppreference.com/w/cpp/thread/sleep_for

https://thispointer.com/learning-shared_ptr-part-1-usage-details/

https://en.cppreference.com/w/cpp/memory/shared_ptr

https://www.geeksforgeeks.org/use-stdthis_threadsleep_for-method-to-sleep-in-cpp/

https://en.cppreference.com/w/cpp/thread/sleep_for

# Shared_ptr

```cpp
#include <chrono>
#include <iostream>
#include <memory>
#include <mutex>
#include <thread>
 
using namespace std::chrono_literals;
 
struct Base
{
    Base() { std::cout << "Base::Base()\n"; }
 
    // Note: non-virtual destructor is OK here
    ~Base() { std::cout << "Base::~Base()\n"; }
};
 
struct Derived: public Base
{
    Derived() { std::cout << "Derived::Derived()\n"; }
 
    ~Derived() { std::cout << "Derived::~Derived()\n"; }
};
 
// void print(auto rem, std::shared_ptr<Base> sp)
void print(auto rem, std::shared_ptr<Base> const& sp)
{
    std::cout << rem << "\n\tget() = " << sp.get()
              << ", use_count() = " << sp.use_count() << '\n';
}
 
void thr(std::shared_ptr<Base> const&  p)
// void thr(std::shared_ptr<Base> p)
{
    std::this_thread::sleep_for(987ms);
    {
        static std::mutex io_mutex;
        std::lock_guard<std::mutex> lk(io_mutex);
        std::cout <<'\n'<< "Testcount:"<< p.use_count() << '\n';
    }
    
    // std::shared_ptr<Base> lp = p; // thread-safe, even though the
                                  // shared use_count is incremented
    {
        static std::mutex io_mutex;
        std::lock_guard<std::mutex> lk(io_mutex);
        // print("Local pointer in a thread:", lp);
        print("Local pointer in a thread:", p);
    }
}
 
int main()
{
    std::shared_ptr<Base> p = std::make_shared<Derived>();
 
    print("Created a shared Derived (as a pointer to Base)", p);
 
    std::thread t1{thr, p}; //, t2{thr, p}, t3{thr, p};
    p.reset(); // release ownership from main
 
    print("Shared ownership between 3 threads and released ownership from main:", p);
 
    t1.join();// t2.join(); t3.join();
 
    std::cout << "All threads completed, the last one deleted Derived.\n";
    return 0;
}

```

## Different pass value method

```cpp
#include <chrono>
#include <iostream>
#include <memory>
#include <mutex>
#include <thread>
 
using namespace std::chrono_literals;
 
struct Base
{
    Base() { std::cout << "Base::Base()\n"; }
 
    // Note: non-virtual destructor is OK here
    ~Base() { std::cout << "Base::~Base()\n"; }
};
 
struct Derived: public Base
{
    Derived() { std::cout << "Derived::Derived()\n"; }
 
    ~Derived() { std::cout << "Derived::~Derived()\n"; }
};

void print(std::shared_ptr<Base> const& sp)
// void print(std::shared_ptr<Base> sp)
{
    std::cout << "\n\tget() = " << sp.get()
              << ", use_count() = " << sp.use_count() << '\n';
}

void thr(std::shared_ptr<Base> const&  p)
{
    std::this_thread::sleep_for(987ms);
    {
        static std::mutex io_mutex;
        std::lock_guard<std::mutex> lk(io_mutex);
        std::cout <<'\n'<< "Testcount:"<< p.use_count() << '\n';
    }
    
    // std::shared_ptr<Base> lp = p; // thread-safe, even though the
                                  // shared use_count is incremented
    {
        static std::mutex io_mutex;
        std::lock_guard<std::mutex> lk(io_mutex);
        print( p);
    }
}
 
int main()
{
    std::shared_ptr<Base> p = std::make_shared<Derived>();
    print(p);
    std::thread t1{thr, p}; 
    p.reset(); 
    print(p);
    t1.join();
    std::cout << "All threads completed, the last one deleted Derived.\n";
    return 0;
}

```


1. pass value in print:

> void print(std::shared_ptr<Base> sp)

```shell
quintin@ubuntu:~/Documents/testspace$ g++ -o makeshare makeshare.cpp -std=c++14
quintin@ubuntu:~/Documents/testspace$ ./makeshare 
Base::Base()
Derived::Derived()

        get() = 0x8cbec0, use_count() = 2

        get() = 0, use_count() = 0

Testcount:1

        get() = 0x8cbec0, use_count() = 2
Derived::~Derived()
Base::~Base()
All threads completed, the last one deleted Derived.
```
>  std::thread t1{print, p};

```shell
quintin@ubuntu:~/Documents/testspace$ g++ -o makeshare makeshare.cpp -std=c++14
quintin@ubuntu:~/Documents/testspace$ ./makeshare 
Base::Base()
Derived::Derived()

        get() = 0x2152ec0, use_count() = 2

        get() = 0, use_count() = 0

        get() = 0x2152ec0, use_count() = 1
Derived::~Derived()
Base::~Base()
All threads completed, the last one deleted Derived.
```

2. pass reference in print

> void print(std::shared_ptr<Base> const& sp)

```shell
quintin@ubuntu:~/Documents/testspace$ g++ -o makeshare makeshare.cpp -std=c++14
quintin@ubuntu:~/Documents/testspace$ ./makeshare 
Base::Base()
Derived::Derived()

        get() = 0x7ccec0, use_count() = 1

        get() = 0, use_count() = 0

Testcount:1

        get() = 0x7ccec0, use_count() = 1
Derived::~Derived()
Base::~Base()
All threads completed, the last one deleted Derived.
```
>  std::thread t1{print, p};

```shell
quintin@ubuntu:~/Documents/testspace$ g++ -o makeshare makeshare.cpp -std=c++14
quintin@ubuntu:~/Documents/testspace$ ./makeshare 
Base::Base()
Derived::Derived()

        get() = 0xd13ec0, use_count() = 1

        get() = 0, use_count() = 0

        get() = 0xd13ec0, use_count() = 1
Derived::~Derived()
Base::~Base()
All threads completed, the last one deleted Derived.
```

3. Add more thread.

```cpp
#include <chrono>
#include <iostream>
#include <memory>
#include <mutex>
#include <thread>
 
using namespace std::chrono_literals;
 
struct Base
{
    Base() { std::cout << "Base::Base()\n"; }
 
    // Note: non-virtual destructor is OK here
    ~Base() { std::cout << "Base::~Base()\n"; }
};
 
struct Derived: public Base
{
    Derived() { std::cout << "Derived::Derived()\n"; }
 
    ~Derived() { std::cout << "Derived::~Derived()\n"; }
};

// void print(auto rem, std::shared_ptr<Base> sp)
// void print(auto rem, std::shared_ptr<Base> const& sp)
// {
//     std::cout << rem << "\n\tget() = " << sp.get()
//               << ", use_count() = " << sp.use_count() << '\n';
// }

void print(std::shared_ptr<Base> const& sp)
// void print(std::shared_ptr<Base> sp)
{
      std::this_thread::sleep_for(987ms);
    std::cout << "\n\tget() = " << sp.get()
              << ", use_count() = " << sp.use_count() << '\n';
}

void thr(std::shared_ptr<Base> const&  p)
// void thr(std::shared_ptr<Base> p)
{
    std::this_thread::sleep_for(987ms);
    {
        static std::mutex io_mutex;
        std::lock_guard<std::mutex> lk(io_mutex);
        std::cout <<'\n'<< "Testcount:"<< p.use_count() << '\n';
    }
    
    // std::shared_ptr<Base> lp = p; // thread-safe, even though the
                                  // shared use_count is incremented
    {
        static std::mutex io_mutex;
        std::lock_guard<std::mutex> lk(io_mutex);
        // print("Local pointer in a thread:", lp);
        // print("Local pointer in a thread:", p);
        print( p);
    }
}
 
int main()
{
    std::shared_ptr<Base> p = std::make_shared<Derived>();
 
    // print("Created a shared Derived (as a pointer to Base)", p);
    // print(p);
    std::cout << "\n\tget() = " << p.get() << ", use_count() = " << p.use_count() << '\n';

    std::thread t1{print, p} , t2{print, p}, t3{print, p};
    p.reset(); // release ownership from main
 
    // print("Shared ownership between 3 threads and released ownership from main:", p);
    std::cout << "\n\tget() = " << p.get() << ", use_count() = " << p.use_count() << '\n';

 
    t1.join(); t2.join(); t3.join();
 
    std::cout << "All threads completed, the last one deleted Derived.\n";
    return 0;
}
```
and the result of the print:
```shell
quintin@ubuntu:~/Documents/testspace$ ./makeshare 
Base::Base()
Derived::Derived()

        get() = 0x8f7ec0, use_count() = 1

        get() = 0, use_count() = 0

        get() = 0x8f7ec0, use_count() = 3

        get() = 0x8f7ec0, use_count() = 3

        get() = 0x8f7ec0, use_count() = 3
Derived::~Derived()
Base::~Base()
All threads completed, the last one deleted Derived.
```

This sub experiment shows that the construction of std::thread will ignore the value pass method in the function, and will copy the shared_ptr anyway, so the use_count of p will add 1, whatever the thr or print function use the reference or value of shared_ptr p.

## Back to the Example

```cpp
#include <chrono>
#include <iostream>
#include <memory>
#include <mutex>
#include <thread>
 
using namespace std::chrono_literals;
 
struct Base
{
    Base() { std::cout << "Base::Base()\n"; }
 
    // Note: non-virtual destructor is OK here
    ~Base() { std::cout << "Base::~Base()\n"; }
};
 
struct Derived: public Base
{
    Derived() { std::cout << "Derived::Derived()\n"; }
 
    ~Derived() { std::cout << "Derived::~Derived()\n"; }
};

// void print(auto rem, std::shared_ptr<Base> sp)
void print(auto rem, std::shared_ptr<Base> const& sp)
{
    std::cout << rem << "\n\tget() = " << sp.get()
              << ", use_count() = " << sp.use_count() << '\n';
}

// void print(std::shared_ptr<Base> const& sp)
// // void print(std::shared_ptr<Base> sp)
// {
//       std::this_thread::sleep_for(987ms);
//     std::cout << "\n\tget() = " << sp.get()
//               << ", use_count() = " << sp.use_count() << '\n';
// }

void thr(std::shared_ptr<Base> const&  p)
// void thr(std::shared_ptr<Base> p)
{
    std::this_thread::sleep_for(987ms);
    // {
    //     static std::mutex io_mutex;
    //     std::lock_guard<std::mutex> lk(io_mutex);
    //     std::cout <<'\n'<< "Testcount:"<< p.use_count() << '\n';
    // }
    
    std::shared_ptr<Base> lp = p; // thread-safe, even though the
                                  // shared use_count is incremented
    {
        static std::mutex io_mutex;
        std::lock_guard<std::mutex> lk(io_mutex);
        print("Local pointer in a thread:", lp);
        // print("Local pointer in a thread:", p);
        // print( p);
    }
}
 
int main()
{
    std::shared_ptr<Base> p = std::make_shared<Derived>();
 
    print("Created a shared Derived (as a pointer to Base)", p);
    // print(p);
    // std::cout << "\n\tget() = " << p.get() << ", use_count() = " << p.use_count() << '\n';

    std::thread t1{thr, p} , t2{thr, p}, t3{thr, p};
    p.reset(); // release ownership from main
 
    print("Shared ownership between 3 threads and released ownership from main:", p);
    //std::cout << "\n\tget() = " << p.get() << ", use_count() = " << p.use_count() << '\n';

 
    t1.join(); t2.join(); t3.join();
 
    std::cout << "All threads completed, the last one deleted Derived.\n";
    return 0;
}
```
different versions of the print results:

```shell
quintin@ubuntu:~/Documents/testspace$ ./makeshare 
Base::Base()
Derived::Derived()
Created a shared Derived (as a pointer to Base)
        get() = 0x15a2ec0, use_count() = 1
Shared ownership between 3 threads and released ownership from main:
        get() = 0, use_count() = 0
Local pointer in a thread:
        get() = 0x15a2ec0, use_count() = 5
Local pointer in a thread:
        get() = 0x15a2ec0, use_count() = 4
Local pointer in a thread:
        get() = 0x15a2ec0, use_count() = 2
Derived::~Derived()
Base::~Base()
All threads completed, the last one deleted Derived.
quintin@ubuntu:~/Documents/testspace$ ./makeshare 
Base::Base()
Derived::Derived()
Created a shared Derived (as a pointer to Base)
        get() = 0x1b4aec0, use_count() = 1
Shared ownership between 3 threads and released ownership from main:
        get() = 0, use_count() = 0
Local pointer in a thread:
        get() = 0x1b4aec0, use_count() = 4
Local pointer in a thread:
        get() = 0x1b4aec0, use_count() = 3
Local pointer in a thread:
        get() = 0x1b4aec0, use_count() = 2
Derived::~Derived()
Base::~Base()
All threads completed, the last one deleted Derived.
```
And if the sleep function is placed after lp
```cpp
void thr(std::shared_ptr<Base> const&  p)
// void thr(std::shared_ptr<Base> p)
{
    std::shared_ptr<Base> lp = p; // thread-safe, even though the
                                  // shared use_count is incremented
    std::this_thread::sleep_for(987ms);
    {
        static std::mutex io_mutex;
        std::lock_guard<std::mutex> lk(io_mutex);
        print("Local pointer in a thread:", lp);
        // print("Local pointer in a thread:", p);
        // print( p);
    }
}
```

The result could be:

```shell
quintin@ubuntu:~/Documents/testspace$ g++ -o makeshare makeshare.cpp -std=c++14
quintin@ubuntu:~/Documents/testspace$ ./makeshare 
Base::Base()
Derived::Derived()
Created a shared Derived (as a pointer to Base)
        get() = 0x1e1eec0, use_count() = 1
Shared ownership between 3 threads and released ownership from main:
        get() = 0, use_count() = 0
Local pointer in a thread:
        get() = 0x1e1eec0, use_count() = 6
Local pointer in a thread:
        get() = 0x1e1eec0, use_count() = 4
Local pointer in a thread:
        get() = 0x1e1eec0, use_count() = 2
Derived::~Derived()
Base::~Base()
All threads completed, the last one deleted Derived.

```
# std::enable_shared_from_this

```cpp
#include <iostream>
#include <memory>
 
struct MyObj
{
    MyObj() { std::cout << "MyObj constructed\n"; }
 
    ~MyObj() { std::cout << "MyObj destructed\n"; }
};
 
struct Container : std::enable_shared_from_this<Container> // note: public inheritance
{
    std::shared_ptr<MyObj> memberObj;
 
    void CreateMember() { memberObj = std::make_shared<MyObj>(); }
 
    std::shared_ptr<MyObj> GetAsMyObj()
    {
        // Use an alias shared ptr for member
        return std::shared_ptr<MyObj>(shared_from_this(), memberObj.get());
    }
};
 
#define COUT(str) std::cout << '\n' << str << '\n'
  
#define DEMO(...) std::cout << #__VA_ARGS__ << " = " << __VA_ARGS__ << '\n'
 
int main()
{
    COUT( "Creating shared container" );
    std::shared_ptr<Container> cont = std::make_shared<Container>();
    DEMO( cont.use_count() ); // 1
    DEMO( cont->memberObj.use_count() ); // 0
 
    COUT( "Creating member" );
    cont->CreateMember();
    DEMO( cont.use_count() ); // 1
    DEMO( cont->memberObj.use_count() ); // 1
 
    COUT( "Creating another shared container" );
    std::shared_ptr<Container> cont2 = cont;
    DEMO( cont.use_count() ); // 2
    DEMO( cont->memberObj.use_count() ); // 1
    DEMO( cont2.use_count() ); //2
    DEMO( cont2->memberObj.use_count() ); // 1 ==> SHARE THE SEGMENT WITH CONT
 
    COUT( "GetAsMyObj" );
    std::shared_ptr<MyObj> myobj1 = cont->GetAsMyObj();
    DEMO( myobj1.use_count() ); // 3
    DEMO( cont.use_count() );  //3
    DEMO( cont->memberObj.use_count() ); // 1
    DEMO( cont2.use_count() ); // 3
    DEMO( cont2->memberObj.use_count() ); // 1
    //The reason why cont.use_count() is 3 here is because you are using std::enable_shared_from_this, which is a template class that allows an object to safely generate additional std::shared_ptr instances that share ownership of the object with the original std::shared_ptr. When you call cont->GetAsMyObj(), you are creating a new std::shared_ptr<MyObj> that is an alias of cont, meaning that it shares the same control block and reference count as cont. Therefore, the use_count() of cont, cont2 and myobj1 are all 3, because they are all sharing the same object.
    //Because cont->memberObj and cont2->memberObj are not aliases of each other, they are separate std::shared_ptr<MyObj> instances that own the same MyObj object. When you call cont->CreateMember(), you are creating a new std::shared_ptr<MyObj> and assigning it to cont->memberObj. This std::shared_ptr<MyObj> has its own control block and reference count, which is independent of the std::shared_ptr<Container> that owns cont. When you copy cont to cont2, you are not copying the memberObj, you are only copying the Container object and its pointer to memberObj. Therefore, the use_count() of memberObj is 1, because there is only one std::shared_ptr<MyObj> that owns it.
    //Because the function GetAsMyObj() is not returning a std::shared_ptr<MyObj> that owns memberObj, it is returning an alias of cont that points to memberObj. The alias std::shared_ptr<MyObj> is constructed from two arguments: the std::shared_ptr<Container> returned by shared_from_this() and the raw pointer to memberObj. This means that the alias std::shared_ptr<MyObj> does not increment the reference count of memberObj, it only increments the reference count of cont. The alias std::shared_ptr<MyObj> will keep cont alive as long as it exists, but it will not affect the lifetime of memberObj. Therefore, the use_count() of memberObj is not changed by calling GetAsMyObj().

    COUT( "Copying alias obj" ); 
    std::shared_ptr<MyObj> myobj2 = myobj1;
    DEMO( myobj1.use_count() ); // 4
    DEMO( myobj2.use_count() ); // 4 
    DEMO( cont.use_count() );  // 4
    DEMO( cont->memberObj.use_count() ); // 1 
    DEMO( cont2.use_count() ); // 4 
    DEMO( cont2->memberObj.use_count() ); // 1
 
    COUT( "Resetting cont2" );
    cont2.reset();
    DEMO( myobj1.use_count() ); // 3 
    DEMO( myobj2.use_count() ); // 3
    DEMO( cont.use_count() ); // 3
    DEMO( cont->memberObj.use_count() ); // 1
 
    COUT( "Resetting myobj2" );
    myobj2.reset();
    DEMO( myobj1.use_count() ); // 2
    DEMO( cont.use_count() ); // 2
    DEMO( cont->memberObj.use_count() ); // 1
 
    COUT( "Resetting cont" );
    cont.reset(); 
    DEMO( myobj1.use_count() ); // 1
    DEMO( cont.use_count() ); // 0
}
```