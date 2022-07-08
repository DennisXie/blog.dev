---
author: "Dennis Xie"
title: "Create Python Bindings with Cython"
date: 2022-07-08T13:01:16+08:00
description: "The basic usages of Cython to create Python bindings"
draft: false
tags: ["python"]
categories: ["tech"]
postid: "7fa8acd353713dd9afc254459d49a009211f7063"
---
# Background

## Foreign function interface

As we know, many operating systems only provide C API for system calls.  This statement is not quite correct, but we assume this is right because we usually interact with operating systems via C API directly or indirectly. Have you ever thought about that why non-C language can interact with operating systems? Before we discuss this question, we need to know some knowledge of foreign function interface (FFI). 

The term FFI refers to the language features for inter-language calls. Some of the non-C languages interact with operating systems via FFI. Many languages refer their FFI as ‘language bindings’, such as Python bindings. Some languages have their own terminology, such as Java’s JNI.

## Python bindings

Python bindings allows you to call C API in pure Python or run Python scripts in C program.

There are two basic ways to implement the Python Bindings. 

- The first way is using ctypes, a library provided by Python.
- The second way is using Python/C API, a library provided by CPython.

We can also choose to use some 3rd-party tools and libraries to make our life easier.

# Create Python bindings via Cython

## Introduction

Cython is a programming language that makes writing C extensions for the Python language as easy as Python itself. It has a compiler that can compile Python and Cython to C. Cython is a powerful tool that can provide you a deep level of control when creating Python bindings for C and C++. It can let you write Python-esque code that manually controls the GIL. It can also let you use standard C/C++ library. You can easily install Cython with the following command.

```bash
pip install Cython
```

## Basic usages of Cython

### The simplest example

Any valid Python code is valid Cython code. This is the simplest example. Save the following code in the *hello.pyx*.

```python
# hello.pyx
def say_hello_to(name):
    print("Hello %s" % name)
```

Now, we can start to compile and build our extension. Save the following code in the *setup.py*.

```python
# setup.py
from setuptools import setup
from Cython.Build import cythonize

setup(
    ext_modules = cythonize("hello.pyx")
)
```

Use the following command to build the Cython file. We can only use this module in the *setup.py*’s directory because we didn’t install this module.

```bash
python setup.py build_ext --inplace
```

We can use this Cython module now! Just open the python interpreter and simply import it as if it was a regular Python module.

```python
Python 3.6.9 (default, Oct  8 2020, 12:12:24)
[GCC 8.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import hello
>>> hello.sayHelloTo("Cython")
Hello Cython
```

### Using static types

In Cython, we can use static type to improve the performance in some cases. We can use cdef to declare static types. We can declare not only basic types of C but also struct, union, enum and their pointer types.

```python
def fibo(n):
    cdef int i = 2
    cdef int a = 0
    cdef int b = 1
    cdef int c = 0

    if n == 0:
        return a
    if n == 1:
        return b

    while i <= n:
        c = a + b
        a = b
        b = c
        i += 1
    return b
```

### Function definitions

There are three ways to define a function in Cython. We can use def, cdef, and cpdef to define a function. Functions with def statement are Python functions. They take Python objects as parameters and return Python objects. Functions with cdef statement are C functions. They take either Python objects or C values as parameters and can return either Python objects or C values. Functions with cpdef statement is a kind of hybrid function. They use the faster C calling convention when being called from other Cython code. Be careful, only functions with def or cpdef statement can be called from Python code. The following table shows their supported features.

| statement | Python objects param/return | C variable param/return | Called from python |
| :-: | :-: | :-: | :-: |
| def |   √ |   √ |  ×  |
| cdef |  √ |   √ |  √  |
| cpdef | √ |   √ |  √  |

The following code block shows the usage of these statements. When you need to call a C function from Python, you can use def or cpdef to write a wrapper function to call that C function.

```python
cdef double calcExponent(int a, int x):
    cdef int i = 0 
    cdef int flag = 0
    cdef double ans = 1.0

    if x == 0:
        return 1
    
    if x < 0:
        flag = 1
        x = -x

    while i < x:
        ans = ans * a
        i += 1
    
    if flag > 0:
        ans = 1 / ans
    else:
        return ans

def pyCalcExponent(int a, int x):
    return calcExponent(a, x)

cpdef double cpCalcExponent(a, x):
    return calcExponent(a, x)
```

## Call C standard library

There are some pre-defined Cython modules for libc, libcpp and posix in Cython package. You can find the complete list of these modules [here](https://github.com/cython/cython/tree/master/Cython/Includes). The following code block shows the usage of pre-defined modules.

```python
# foolib.pyx
from libc.math cimport cos

def pyCos(x):
    return cos(x)
```

The libc math library is special in that it is not linked by default on some Unix-like systems. So we need to configure the build system to link the shared library m.

```python
# setup.py
from setuptools import setup, Extension
from Cython.Build import cythonize

setup(
    ext_modules=cythonize([
        Extension("foolib", sources=["foolib.pyx"], libraries=["m"])
        ]),
    zip_safe=False,
)
```

The following command is the building command.

```bash
# build & install command
python setup.py build_ext --inplace
```

```python
Python 3.6.9 (default, Oct  8 2020, 12:12:24)
[GCC 8.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import foolib as f
>>> f.pyCos(3.14159265357589)
-1.0
```

## Build with C code

### The C code

In this example, we will implement two functions in C. The function called *fibo* calculates the Nth Fibonacci number. The other function called *calcDistance* calculates the distance of two points. The point and distance are defined in the header file *foolib.h* and the two functions are implemented in the source file *foolib.c*. Here is the header file and source file of the foolib.

```c
// foolib.h
#ifndef __FOOLIB_H
#define __FOOLIB_H
typedef struct _Point {
	  char name[50];    // Define a char array for name instead of char* to avoid memory management.
    int x;
    int y;
} Point;

typedef struct _Distance {
    char start_name[50];
    char end_name[50];
    double distance;
} Distance;

extern int fibo(int n);

extern Distance calcDistance(Point a, Point b);
#endif
```

```c
// foolib.c
#include <string.h>
#include <math.h>

#include "foolib.h"

int fibo(int n) {
    int a = 0;
    int b = 1;
    if (n == 0) {
        return a;
    }
    if (n == 1) {
        return b;
    }
    int i;
    int c;
    for (i = 2; i <= n; i++) {
        c = a + b;
        a = b;
        b = c;
    }
    return b;
}

Distance calcDistance(Point a, Point b) {
    Distance distance;
    strcpy(distance.start_name, a.name);
    strcpy(distance.end_name, b.name);
    int x = b.x - a.x;
    int y = b.y - a.y;
    double d2 = (double) (x * x + y * y);
    distance.distance = sqrt(d2);
    return distance;
}
```

Then we compile the source code into the shared library with the following commands.

```bash
	gcc -c -Wall -Werror -fpic foolib.c
	gcc -shared -o libfoolib.so foolib.o -lm
```

In the first line, we compile the source code into the position-independent code (PIC)<sup>[pic](#explanation)</sup> and we can get an object file named *foolib.o*. In the second line, we turn this object file into the shared library. Be careful, we need to name the shared library in the format of *lib{name}.so*<sup>[soname](#explanation)</sup>. So, in this example, we need to name the shared library _libfoolib.so_.

### The Cython code

Now, we start to write the Cython code. We first create a file called *pfoolib.pxd*. In the pxd file, we import the *foolib.h* header file and define the struct and function in Cython. Because Python code can’t call the functions or use the structures defined in the pxd file. So, we need to define wrapper functions to call these C functions and wrapper classes can be used by Python code. We define the wrapper functions and classes in the *pfoolib.pyx*. We can simply regard the pyd file as the header file of the pyx file. But we only declare the things implemented by C. The following code blocks are the pxd file and pyx file of the pfoolib.

```python
# pfoolib.pxd
cdef extern from "foolib.h":
    # We can just define the fields that we need to use.
    # If we don't need to use any filed of this struct in some cases,
    # we can just put pass in the body of struct declaration.
    ctypedef struct Point:
        char name[50]
        int x
        int y
    
    ctypedef struct Distance:
        char start_name[50]
        char end_name[50]
        double distance
    
    int fibo(int n)

    Distance calcDistance(Point a, Point b)
```

In the pxd file, we can just declare the fields that we need to use. If we don’t need any field of this struct, we can just put pass in the body of struct declaration.

```python
# pfoolib.pyx
cimport pfoolib

from libc.string cimport strcpy, strlen

# The class name can't be same as Point defined in pfoolib.pxd, so we name it PyPoint
cdef class PyPoint:
    cdef public bytes name
    cdef public int x
    cdef public int y

    def __init__(self, str name, int x, int y):
				# We need to encode the string into byte array.
        self.name = name.encode("utf-8")
        self.x = x
        self.y = y
    
    cdef pfoolib.Point toCPoint(self):
        cdef pfoolib.Point p
        # Copy the byte array to the char array. 
        strcpy(p.name, self.name)
        p.x = self.x
        p.y = self.y
        return p

cdef class PyDistance:
    cdef public str start_name
    cdef public str end_name
    cdef public double distance

    def __init__(self, char* start_name, char* end_name, float distance):
        self.start_name = start_name[:strlen(start_name)].decode("utf-8")
        self.end_name = end_name[:strlen(end_name)].decode("utf-8")
        self.distance = distance
    
    cdef pfoolib.Distance toCDistance(self):
        cdef pfoolib.Distance d
        d.start_name = self.start_name
        d.end_name = self.end_name
        d.distance = self.distance

def pyFibo(n):
		return fibo(n)

cpdef PyDistance pyCalcDistance(PyPoint a, PyPoint b):
		# Transfer the python class to c struct.
    cdef pfoolib.Point cpa = a.toCPoint()
    cdef pfoolib.Point cpb = b.toCPoint()
    cdef pfoolib.Distance dis = pfoolib.calcDistance(cpa, cpb)
    d = PyDistance(dis.start_name, dis.end_name, dis.distance)
    return d
```

If the pyx file, there are a few things we need to be careful with.

- The class name needs to be different from the name defined in the pxd file. In the official document, they use the same name. But when I use the same name, I got a redeclaration error and I don’t know why.
- Because we use the char array for name, we can only use strcpy to copy the string to the name. If you just use assignment, you will get a runtime index error. If you define the name as char*, then you can use assignment directly.
- The bytes in Python3 is correspond to the string in C. So, we need to use encode and decode to transform the string.
- Cython compiler will generate a same name C file for every pyx file. If your C files and pyx files are in the same directory, you’d better give them different names.

The following file is the setup file of our module. We can build and install our module with this file. Pay attention to the *libraries* field of the Extension class, this field tells Cython what library we need. Here we provide the *foolib*, then the C compiler will find the *libfoolib.so* in the link stage.

```python
# setup.py
from setuptools import setup, Extension
from Cython.Build import cythonize

setup(
    name = "pfoolib",
    ext_modules=cythonize([
        Extension("pfoolib", ["pfoolib.pyx"], libraries=["foolib"])
    ]),
    zip_safe=False
)
```

Now, works are almost done. We just type the following command to build the Cython module.

```bash
CFLAGS="-I/path/to/headers" LDFLAGS="-L/path/to/library" python3 setup.py build_ext --inplace
```

We can use the foolib shared library in Python now. In the build directory (or you use it anywhere if you install it), type python and try it in the terminal.

```python
Python 3.6.9 (default, Oct  8 2020, 12:12:24)
[GCC 8.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import pfoolib
>>> a = pfoolib.PyPoint("aaa", 1, 3)
>>> b = pfoolib.PyPoint("ccc", 2, 3)
>>> c = pfoolib.pyCalcDistance(a, b)
>>> c.start_name
'aaa'
>>> c.end_name
'bbb'
>>> c.distance
1.0
```

### The makefile

We can write a simple Makefile to avoid typing the long shell command. The following file is the makefile.

```makefile
# Makefile
pwd:=$(shell pwd)

build: sharelib
	CFLAGS="-I$(pwd)" LDFLAGS="-L$(pwd)" python3 setup.py build_ext --inplace

sharelib:
	gcc -c -Wall -Werror -fpic foolib.c
	gcc -shared -o libfoolib.so foolib.o -lm

clean:
	rm -f foolib.o libfoolib.so pfoolib.*.so
	rm -rf build
```

Now, we can simply use the make command to compile, build and clean.

```bash
make sharelib    # compile the share library with out building Cython module
make build       # compile the share library and build Cython module
make clean       # remove the compiling and building results.
```

## Python callback functions

Sometimes, we need to provide callbacks to our library. This section will give a simple example to show how to provide callback functions to our C library. In this example, we implement a function to filter an array’s elements. The function will take an array and predicate function for input. 

### The C code

The following code blocks show the header file and source file.

```c
// foolib.h
#ifndef __FOOLIB_H
#define __FOOLIB_H
typedef int (*predicate)(int, void*);

extern int filter(int [], int, predicate, void*);
#endif
```

In the header file, we declare the predicate type for callback functions. The predicate has a void* parameter to get the context. The context will contain the real Python predicate function. If we don’t provide a void* parameter, we can only use a callback function without any context.

```c
// foolib.c
int filter(int arr[], int length, predicate check, void* p) {
    int i = 0;
    int j = 0;
    for (i = 0; i < length; i++) {
        if (check(arr[i], p) > 0) {
            arr[j] = arr[i];
            j++;
        }
    }
    return j;
}
```

### The Cython code

The following code blocks show the pxd file and pyx file.

```python
# pfoolib.pxd
cdef extern from "foolib.h":
		int filter(int [], int, predicate, void*)
```

```python
# pfoolib.pyx
cimport pfoolib
from libc.stdlib cimport malloc, free

cdef int cCallBack(int a, void *p):
    try:
        func = <object>p
        if func(a):
            return 1
        else:
            return 0
    except:
        return -1

def pyFilter(list arr, judge):
    cdef int* cArr = <int*> malloc(len(arr) * sizeof(int))
    if not cArr:
        raise MemoryError()

    try:
        for i in range(len(arr)):
            cArr[i] = arr[i]

        newLength = filter(cArr, len(arr), cCallBack, <void*> judge)

        for i in range(newLength):
            arr[i] = cArr[i]
        return arr[:newLength]
    finally:
        free(cArr)
```

The *cCallBack* is a callback function provided to the filter function. *cCallBack* will get the Python callback function from void* parameter and call it to check whether to keep an element in the array. The filter function takes an integer array. The array is equivalent to the pointer in C. Cython can’t automatically transfer a Python list to an array if the array doesn’t have a fixed length. So, we need to malloc an array in the heap, copy the elements to this array, and finally free this array after use.

The setup script and build command are the same as the previous example. Now, try to use this filter function.

```python
Python 3.6.9 (default, Oct  8 2020, 12:12:24)
[GCC 8.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import pfoolib as p
>>> p.pyFilter([1, 3, -3, 0, 9, -8, 2], lambda x: x > 0)
[1, 3, 9, 2]
```
# Conclusion
The examples in this blog only show the basic usages of Cython. But these examples can cover many situations that integrate Python code with C code. In the future, I'll introduce more advanced usages of Cython.

# Explanation

1. What is position independent code (PIC)? PIC is code that works no matter where in memory it is placed. Because several different programs can all use one instance of your shared library, the library cannot store things at fixed addresses, since the location of that library in memory will vary from program to program.<sup>2</sup>
2. Every shared library in Unix-like system has a ‘so-name’, the so-name need to have a lib prefix and .so suffix by default. So, the so-name of foo library is libfoo.so. When you pass a '-lfoo' parameter to GCC, the compiler will look up the libfoo.so in the link stage. But some basic C libraries don’t have this constraint.

# Reference

1. [Cython official document](https://cython.readthedocs.io/en/latest/index.html)
2. [Shared libraries with GCC on Linux](https://www.cprogramming.com/tutorial/shared-libraries-linux-gcc.html#fn:pic)
3. [Makefile tutorial](https://makefiletutorial.com/)