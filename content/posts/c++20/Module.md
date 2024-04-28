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

1. 由于 `#include` 属于预处理指令，它简单地将头文件的内容复制到当前文件，因此，如果忘记做头文件保护，会出现重定义（redefinition）错误；
2. 头文件不需要 self-contained，如下例：

    ```cpp
    // File: a.h
    class A { std::string a; };

    // File: main.cpp
    #include <string>
    #include <a.h>

    int main() {}
    ```

    这仍然可以编译，但如果将该头文件提供给外部库使用，如果该库在引入该头文件之前未引入`<string>`，编译将会失败；
3. 多次引入过大的头文件将显著降低编译效率，每一次引入该头文件，`#include` 都会将其中的内容完全复制过来。

正是为了解决以上问题，C++ 引入了 *Modules*。

## 声明模块

```cpp
// syntax
[export] module <module_name> [module_partition] [attr];
```

其中，`module_name` 为模块的名称（必选），`module_partition` 为模块分块名称（可选），`attr` 为模块属性（可选）。

声明 `export` 时，该源文件为声明文件，否则，该源文件为实现文件。

### 导出变量、函数、类或模版

声明模块后，就可以在该源文件中导出变量、函数或类

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

export template<typename T>
T max(T a, T b);

export template<typename T>
class TemplatedType;
```

### 分离声明和实现

Modules 支持分离声明和实现，声明文件被称为 `primary module interface unit`，实现文件被称为 `module implementation unit`。例如：

```cpp
// primary module interface unit for A
export module A;
export const char* hello();

// module implementation unit for A
module A;
const char* hello() { return "Hello!"; }
```

### 模块分块（主要目的是逻辑上的分离）

```cpp
// syntax
export module A:B; // Declares a module interface unit for module 'A', partition ':B'.

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

### 如何引入头文件

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

### 私有模块声明

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

## 使用模块

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

## 模块所有权问题

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

## 性能比较

## Reference

1. [Modules](https://en.cppreference.com/w/cpp/language/modules)
2. [Cppcon 2023](https://www.youtube.com/watch?v=_x9K9_q2ZXE&t=1633s)
