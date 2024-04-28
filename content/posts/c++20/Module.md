+++
title = "C++ 20 模块简介"
date = "2024-04-28T11:26:10+08:00"
author = "Civitasv"
authorTwitter = "" #do not include @
cover = ""
tags = ["C++", "Module"]
keywords = ["Module", "dependency"]
description = "介绍 C++20 引入的 Module 的使用方法"
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

## 背景

模块（Modules）是 C++ 20 引入的，类似于头文件（header），用于在不同源文件之间共享变量、函数和类的声明或实现，从而重用代码。

在模块未被引入之前，我们一般采用头文件，然而，这种方式存在以下几个问题：

## 如何用使

### 声明一个模块

```cpp
// syntax
[export] module <module_name> [module_partition] [attr];
```

其中，`module_name` 为模块的名称（必选），`module_partition` 为模块分区（可选），`attr` 为模块属性（可选）。

`声明 export` 时，该源文件为声明文件，否则，该源文件为实现文件。

模块中建议不要使用 `#include` 引入头文件，可以使用：

```cpp
// syntax
[export] import <header_name> [attr]

/////// A.cpp (primary module interface unit of 'A')
export module A;
 
import <iostream>;
export import <string_view>;
 
export void print(std::string_view message)
{
    std::cout << message << std::endl;
}
 
/////// main.cpp (not a module unit)
import A;
 
int main()
{
    std::string_view message = "Hello, world!";
    print(message);
}

```

但如果一定要使用 `#include` 的话，可以使用全局模块（Global module fragment）。

```cpp
// syntax
module;

// example
/////// A.cpp (primary module interface unit of 'A')
module;
 
// Defining _POSIX_C_SOURCE adds functions to standard headers,
// according to the POSIX standard.
#define _POSIX_C_SOURCE 200809L
#include <stdlib.h>
 
export module A;
 
import <ctime>;
 
// Only for demonstration (bad source of randomness).
// Use C++ <random> instead.
export double weak_random()
{
    std::timespec ts;
    std::timespec_get(&ts, TIME_UTC); // from <ctime>
 
    // Provided in <stdlib.h> according to the POSIX standard.
    srand48(ts.tv_nsec);
 
    // drand48() returns a random number between 0 and 1.
    return drand48();
}
 
/////// main.cpp (not a module unit)
import <iostream>;
import A;
 
int main()
{
    std::cout << "Random value between 0 and 1: " << weak_random() << '\n';
}
```

Private module fragment:

```cpp
// syntax
module : private;

// example
export module foo;
 
export int f();
 
module : private; // ends the portion of the module interface unit that
                  // can affect the behavior of other translation units
                  // starts a private module fragment
 
int f()           // definition not reachable from importers of foo
{
    return 42;
}
```

Module partitions:

```cpp
// syntax
export  module A:B; // Declares a module interface unit for module 'A', partition ':B'.

// example

/////// A-B.cpp   
export module A:B;
...
 
/////// A-C.cpp
module A:C;
...
 
/////// A.cpp
export module A;
 
import :C;
export import :B;
 
...
```

### 导出变量、函数或类 [^1]

声明模块后，就可以在该源文件中导出变量、函数或类。

```cpp
// syntax
export declaration
export { declaration-seq }

// example
export module A; // declares the primary module interface unit for named module 'A'
 
// hello() will be visible by translations units importing 'A'
export char const* hello() { return "hello"; } 
 
// world() will NOT be visible.
char const* world() { return "world"; }
 
// Both one() and zero() will be visible.
export
{
    int one()  { return 1; }
    int zero() { return 0; }
}
 
// Exporting namespaces also works: hi::english() and hi::french() will be visible.
export namespace hi
{
    char const* english() { return "Hi!"; }
    char const* french()  { return "Salut!"; }
}
```

### 引入一个模块

```cpp
[export] import <module_name> [attr]
```

假设在另一个源文件中声明了模块 B: `export module B`，如果想让所有引入 B 的也能访问模块 A，可以使用 `export import A`。

```cpp
/////// A.cpp (primary module interface unit of 'A')
export module A;
 
export char const* hello() { return "hello"; }
 
/////// B.cpp (primary module interface unit of 'B')
export module B;
 
export import A;
 
export char const* world() { return "world"; }
 
/////// main.cpp (not a module unit)
#include <iostream>
import B;
 
int main()
{
    std::cout << hello() << ' ' << world() << '\n';
}
```

## 模块所有权问题 [^1]

The following declarations are not attached to any named module (and thus the declared entity can be defined outside the module):

* [namespace](https://en.cppreference.com/w/cpp/language/namespace "cpp/language/namespace") definitions with external linkage;
* declarations within a [language linkage](https://en.cppreference.com/w/cpp/language/language_linkage "cpp/language/language linkage") specification.

```cpp
export module lib_A;
 
namespace ns // ns is not attached to lib_A.
{
    export extern "C++" int f(); // f is not attached to lib_A.
           extern "C++" int g(); // g is not attached to lib_A.
    export              int h(); // h is attached to lib_A.
}
// ns::h must be defined in lib_A, but ns::f and ns::g can be defined elsewhere (e.g.
// in a traditional source file).
```

## Reference

[^1]: [Modules](https://en.cppreference.com/w/cpp/language/modules)
