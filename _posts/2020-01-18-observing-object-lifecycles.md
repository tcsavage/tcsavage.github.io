---
title: Observing Object Lifecycles in C++
tags: programming c++
---

# Being lazy

I'm an incredibly lazy programmer. If the compiler can do something for me, I'll go out of my way to _make sure_ it does. To that end, I prefer to let the compiler take care of implementing constructors, assignment operators, and destructors whenever I can. I just can't be bothered with all that.

To wit: C++ has some [fairly complex rules](https://en.wikipedia.org/wiki/Special_member_functions) which control when certain constructors or assignment operators are implicitly declared by the compiler. This has resulted in a common [rule of thumb](https://en.wikipedia.org/wiki/Rule_of_three_(C++_programming)#Rule_of_Five) which essentially says that if you implement your own destructor, a copy/move constructor or copy/move assignment operators, you should probably implement all five.

That's a lot of fairly repetitive and tedious work. So I much prefer to follow the rule of zero: if you don't need to do anything specific, implement none of the above, rely on RAII, and let the compiler do all the work.

However, I'm also fairly new to C++ and have often wanted to trace the lifetimes of certain objects and observe when they destruct. My first instinct when trying to do this was to define a destructor and have it log something. But this inhibits implicit declaration of copy/move constructors and assignment operators. What I wanted was to encapsulate the pattern, make it reusable and easy to drop-in and take-out again.

# LogOnDestructed

So here's the idea, create a class with a destructor which logs something when it runs. An instance can then be declared as the first member of a class and RAII will run that destructor last.

```cpp
// This template class can be specialised to let the programmer specify
// human-readable names for classes.
template <typename T>
class TypeName {
public:
    static const char *get() {
        return typeid(T).name();
    }
};

template <typename T>
class LogOnDestructed {
public:
    // Constructors/assignent operators for completeness.
    LogOnDestructed() {}
    LogOnDestructed(const LogOnDestructed&) {}
    LogOnDestructed(LogOnDestructed&&) noexcept {}
    LogOnDestructed& operator=(const LogOnDestructed& other) {
        return *this = LogOnDestructed(other);
    }
    LogOnDestructed& operator=(LogOnDestructed&& other) noexcept {
        return *this;
    }

    ~LogOnDestructed() {
        std::cerr << TypeName<T>::get() << " object destructed\n";
    }
};
```

Here's an example of how you might use it:

```cpp
class MyClass {
    LogOnDestructed<MyClass> _logOnDestructed;
};

template <> class TypeName<MyClass> {
public:
    static const char *get() {
        return "MyClass";
    }
};
```

# Taking it further

You can take this concept further and create a version which logs in the constructor as well if you care to observe when the containing class is constructed.

```c++
template <typename T>
class LogLifecycle {
public:
    LogLifecycle() {
        std::cerr << TypeName<T>::get() << " object constructed\n";
    }
    LogLifecycle(const LogLifecycle&) {}
    LogLifecycle(LogLifecycle&&) noexcept {}
    LogLifecycle& operator=(const LogLifecycle& other) {
        return *this = LogLifecycle(other);
    }
    LogLifecycle& operator=(LogLifecycle&& other) noexcept {
        return *this;
    }

    ~LogLifecycle() {
        std::cerr << TypeName<T>::get() << " object destructed\n";
    }
};
```

# It's in my toolbox

I've found this a handy tool to keep around in my C++ toolbox simply because of how easy it is to deploy. It's helped me solve a few issues I encountered recently, and while it's certainly not the only way I could have debugged those issues (and probably not the best way either) it definitely got the job done.
