---
title: "C++ new那些事儿"
description: 
categories:
  - "C_C++"
tags:
  - "C++"
date: 2024-09-30T09:52:09+08:00
slug: "things-about-new"
math: 
license: 
hidden: false
draft: false
---

# new expression

分配内存-调用构造函数-返回指针

The new operator is an operator which denotes a request for memory allocation on the Heap. If sufficient memory is available, new operator initializes the memory and returns the address of the newly allocated and initialized memory to the pointer variable. When you create an object of class using new keyword(normal new).

- The memory for the object is allocated using operator new from heap.
- The constructor of the class is invoked to properly initialize this memory.

# operator new

只分配内存-返回指针/抛出异常

Operator new is a function that allocates raw memory and conceptually a bit similar to malloc().

- It is the mechanism of overriding the default heap allocation logic.
- It doesn’t initializes the memory i.e constructor is not called. However, after our overloaded new returns, the compiler then automatically calls the constructor also as applicable.
- It’s also possible to overload operator new either globally, or for a specific class


# New operator vs operator new

- Operator vs function: new is an operator as well as a keyword whereas operator new is only a function.
- New calls “Operator new”: “new operator” calls “operator new()” , like the way + operator calls operator +()
- “Operator new” can be Overloaded: Operator new can be overloaded just like functions allowing us to do customized tasks.
- Memory allocation: ‘new expression’ call ‘operator new’ to allocate raw memory, then call constructor.

# References
> 1. 
