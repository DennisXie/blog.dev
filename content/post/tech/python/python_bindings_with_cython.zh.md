---
author: "Dennis Xie"
title: "使用Cython实现Python Bindings"
date: 2022-08-10T21:30:26+08:00
description: "Cython的简单入门使用"
draft: false
tags: ["python"]
categories: ["tech"]
postid: "7fa8acd353713dd9afc254459d49a009211f7064"
---
# 背景

## 语言交互接口(FFI)

一般操作系统会提供系统调用的C API。这种说法其实并不正确，但是我们假设这个说法是正确的。毕竟很多Linux发行版都是自带了GCC编译器的，我们就简单的认为提供了系统调用的C API。然后我们思考一下这么一个问题，一个非C语言是怎么实现和操作系统交互的？主要有两种方式实现，一种是使用中断来实现系统调用，另外一种方式是间接使用C语言的系统调用API。

间接调用C语言的API涉及到了语言交互接口(FFI, Foreign Function Interface)的概念。不少编程语言就通过使用FFI来实现系统调用。很多语言把FFI叫做"Language Bindings"，比如Python Bindings。还有一些语言有自己的叫法，比如Java的JNI。通过使用FFI，像Python这些语言就可以比较简单的实现系统调用。

## Python Bindings

Python Bindings可以让Python代码调用C API，或者在C程序中运行Python脚本。实现Python Bindings有两种基本的方式，分别如下：

- 使用Python的标准库ctypes
- 使用CPython提供的库Python/C API

和很多基础库一样，这两个库都很底层，在业务代码中使用起来会比较复杂。我们可以用一些封装好的三方库来实现Python Bindings，比如使用Cython。

# 使用Cython实现Python Bindings

## Cython简介

Cython是一种能够方便为Python编写C扩展的编程语言，其编译器可以将Cython代码编译为C代码。Cython兼容Python的所有语法且能编译成C，故Cython可以允许开发者通过Python式的代码来手动控制GIL。Cython其语法也允许开发者方便的使用C/C++代码库。通过以下命令即可安装Cython。
```bash
pip install Cython
```

## Cython基础使用方法

### Hello, World!
所有合法的Python语法都是合法的Cython语法，我们先写一个简单的例子保存到*hello.pyx*。

```python
# hello.pyx
def say_hello_to(name):
    print("Hello %s" % name)
```

接着写一下setup文件来编译和构建这个扩展。

```python
# setup.py
from setuptools import setup
from Cython.Build import cythonize

setup(
    ext_modules = cythonize("hello.pyx")
)
```

使用如下命令构建这个Cython文件。

```bash
python setup.py build_ext --inplace
```

接下来我们就可以打开Python解释器并引入和使用这个Cython扩展了。

```python
Python 3.6.9 (default, Oct  8 2020, 12:12:24)
[GCC 8.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import hello
>>> hello.sayHelloTo("Cython")
Hello Cython
```

### 静态类型

Cython可以允许我们在代码中定义静态来类型来提升程序的性能。我们可以使用cdef关键字来定义静态类型。cdef还可以用于定义struct, union, enum以及指针类型等。

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

### 定义函数

我们可以在Cython代码中使用def, cdef和cpdef这三个关键字来定义函数。通过def定义的函数是Python函数，这类函数的参数和返回类型都是Python对象。通过cdef定义的函数是C函数，这类函数的参数和返回类型可以是Python对象，也可以是C变量。cpdef是cdef和def的混合体，Cython代码在调用这类函数的时候会使用比较快的C调用约定(calling convention)。另外需要注意的是只有def和cpdef定义的函数可以被Python代码调用。下表展示了这几种关键字的不同特性。

| 关键字 | Python objects为参数和返回值 | C 变量为参数和返回值 | Python代码可访问 |
| :-: | :-: | :-: | :-: |
| def |   √ |   × |  √  |
| cdef |  √ |   √ |  ×  |
| cpdef | √ |   √ |  √  |

下方代码展示了三种关键字的用法。当我们需要使用Python代码访问C函数的时候，我们可以使用def或者cpdef关键字定义一个包装函数来调用C函数。

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

## 调用C标准库函数示例

Cython包中预定义了libc, libcpp和posix的函数可以直接使用。完整的列表可以戳[这里](https://github.com/cython/cython/tree/master/Cython/Includes)。下面的代码展示了如何使用这些预定义的模块。

```python
# foolib.pyx
from libc.math cimport cos

def pyCos(x):
    return cos(x)
```

libc的math库在某些类Unix系统上默认没有链接，所以需要手动在构建脚本里明确连接共享库m。

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

构建命令如下

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

## 构建及调用C代码示例

前面介绍了直接使用Cython已经封装好的标准库，接下来的例子会介绍使用我们自己写的C/C++库。本例会实现一个C函数*fibo*来计算斐波那契数。另外实现一个函数叫*calcDistance*，用来计算平面直角坐标系中两点的距离。

### C源代码

点和距离的结构体以及函数的定义放在*foolib.h*头文件中，两个函数的实现放在*foolib.c*文件中。

下面是头文件和源文件的代码：

```c
// foolib.h
#ifndef __FOOLIB_H
#define __FOOLIB_H
typedef struct _Point {
	char name[50];    // 这里使用数组来存储名字避免内存管理.
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

接下来我们使用下方的命令把源码编译成共享库。

```bash
	gcc -c -Wall -Werror -fpic foolib.c
	gcc -shared -o libfoolib.so foolib.o -lm
```

第一条命令会把源码编译成位置无关代码(PIC, position-independent code)<sup>[pic](#名词解释)</sup> 并且会生成一个叫*foolib.o*的object文件。第二行命令会将object文件进行链接生成共享库。需要注意的是，共享库需要按*lib{name}.so*<sup>[soname](#名词解释)</sup>的格式命名。本例中，我们需要将共享库命名为_libfoolib.so_.

### Cython代码

接下来是Cython代码。我们先创建一个pxd文件*pfoolib.pxd*。在pxd文件中，我们引入*foolib.h*头文件，然后以Cython语法声明一遍头文件中的结构体和函数。因为Python代码没法直接访问pxd文件中定义的结构体和函数，我们还需要定义一些包装函数来访问C函数，定义一些包装类供Python代码使用。我们在pyx文件*pfoolib.pyx*中定义这些包装函数和包装类。我们可以简单的将pyd文件看作Cython的头文件，只是声明C代码实现的内容即可。接下来的代码块为相关的Cython代码。

```python
# pfoolib.pxd
cdef extern from "foolib.h":
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

在pxd文件中，我们可以只声明一些我们需要的字段即可。如果我们一个字段都不需要，那么写个pass就是了。

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

在pyx文件中，我们需要注意一下几点：

- 类名和pxd中的结构体名称需要不同，否则会收获一个重命名的错误(官方示例使用了同样的名字，但是我试了一下报错了)。
- 为了避免管理内存，我们将name字段声明为了char数组，所以我们只能用strcpy函数来拷贝name字符串。如果直接使用赋值操作的话会遇到"runtime index error"。如果name被声明为char*, 那么我们就可以直接使用赋值操作。
- C中的字符串类型对应Python3中的bytes类型。所以我们需要对字符串进行编解码操作。
- Cython的编译器在编译的时候会为pyx文件产生一个同名的c文件。如果我们的C代码和pyx文件在同一个文件夹下，那么我们需要给他们不同的文件名。 

下面的代码块为setup文件。注意*libraries*参数，这个参数告诉Cython构建这个扩展时我们所需要的共享库。这里我们指明了*foolib*，C编译器会在链接阶段寻找*libfoolib.so*这个共享库。

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

使用如下命令构建扩展：

```bash
CFLAGS="-I/path/to/headers" LDFLAGS="-L/path/to/library" python3 setup.py build_ext --inplace
```

接下来就可以在Python中使用这个共享库了。

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

### Makefile代码

我们可以写要给简单的Makefile来避免敲命令行来编译这么多东西。Makefile内容如下：

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

然后我们就可以使用下面的命令来编译C共享库和Cython模块了。Makefile的用法可以参考引用里面的[Makefile tutorial](https://makefiletutorial.com/)

```bash
make sharelib    # compile the share library with out building Cython module
make build       # compile the share library and build Cython module
make clean       # remove the compiling and building results.
```

## 回调函数示例

接下来的例子会展示为C库提供回调函数的用法。在这个例子里面，我们会提供一个函数来过滤掉数组中的某些元素，该函数会以数组和回调函数作为输入, 根据回调函数的返回结果来选择是否过滤某个元素。

### C源代码

下面的代码展示了C代码的头文件和源文件：

```c
// foolib.h
#ifndef __FOOLIB_H
#define __FOOLIB_H
typedef int (*predicate)(int, void*);

extern int filter(int [], int, predicate, void*);
#endif
```

在头文件中，我们声明了回调函数的类型。回调函数会通过一个void*参数来获取上下文，该上下文会获取真实的Python回调函数。如果不这样干的话，我们是没法使用对象的成员方法或者使用闭包的。

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

### Cython代码

接下来是pxd文件和pyx文件。

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

*cCallBack*是提供给filter函数的回调函数。*cCallBack*会从void*参数中获取Python代码提供的回调函数并进行调用。filter函数会以一个整数数组(和整数指针等效)作为入参。如果我们的C代码没有指定数组的长度，那么Cython不能自动的将Python list转换成数组。所以我们需要在堆中创建一个数组，把list中的元素拷贝进数组中，过滤完以后再把内存释放掉。

构建代码及命令和前例相同就不做展示了。使用方法如下：

```python
Python 3.6.9 (default, Oct  8 2020, 12:12:24)
[GCC 8.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import pfoolib as p
>>> p.pyFilter([1, 3, -3, 0, 9, -8, 2], lambda x: x > 0)
[1, 3, 9, 2]
```

# 总结

本文中的例子都是Cython的一些基本用法。但是这些用法可以涵盖日常使用中的很多场景。日后我还会继续介绍一些Cython的用法。

# 名词解释

1. Position independent code (PIC)：PIC 是一种和内存位置无关的代码。因为不同的程序需要使用相同的共享库，但是共享库在不同进程中的内存位置并不相通。所以，共享库中的变量及函数地址不能固定，只能在程序运行时确认。
2. soname: 类Unix系统中每个共享库都会有一个"so-name"。so-name以"lib"为文件名前缀，".so"为后缀。所以foo共享库的so-name为libfoo.so。当在编译时传入"-l foo"参数给GCC的时候，编译器会在链接阶段需寻找libfoo.so文件。但是某些基础的C/C++库没有这种限制。

# 引用
1. [Cython official document](https://cython.readthedocs.io/en/latest/index.html)
2. [Shared libraries with GCC on Linux](https://www.cprogramming.com/tutorial/shared-libraries-linux-gcc.html#fn:pic)
3. [Makefile tutorial](https://makefiletutorial.com/)
