---
layout: post
title: "C++静态链接符号冲突的几种处理方法"
author: "Yang"
header-style: text
tags:
  - C++ linker
  - symbol conflict
  - g++
  - objcopy
  - 符号重定义
  - static link
  - dynamic link
---

- any list
{:toc}

C++项目中第三方库越来越多，链接时符号冲突的可能性就越来越大。比如项目依赖libA和libB，libA和libB都使用了libX，在链接项目的时候就很可能产生libX的符号冲突，导致链接报错。本文介绍C++链接符号冲突时的几种应对方法。



# allow-multiple-definition

```
--allow-multiple-definition
-z muldefs
Normally when a symbol is defined multiple times, the linker will report a fatal error. These options allow multiple definitions and the first definition will be used.
```


链接器ld有个选项allow-multiple-definition，当有符号重定义时，使用这个选项可以让链接器忽略错误，实际使用的是解析时遇到的第一个定义，后面的符号定义会被忽略。比如可以使用如下命令让项目顺利链接，实际使用的是libA.a中的符号定义。

```g++ -Wl,--allow-multiple-definition main.cpp libA.a libB.a```

这个方法有个缺点。如果libA和libB使用的是不同版本的libX，或者说libA中的libX是经过修改的，那么实际运行的时候，libB也是用的libA中的符号定义，这就可能会出问题。



# objcopy --localize-symbol

C++可重定位目标模块（即.o文件）中有个符号表，包含本模块所有定义和引用的符号信息。符号又分为全局符号和本地符号两种。全局符号指本模块定义的非静态函数和全局变量，其他模块可见，可以供其他模块使用。本地符号指静态函数和静态变量，只能供本模块使用，其他模块不可见。使用 `readelf` 命令可以查看一个符号是本地还是全局。比如 libB.cpp 内容如下：

```
int subfunc(int a, int b) {
    return a - b;
}

int funcB(int a, int b) {
    return subfunc(a, b);
}

static void foo() {
}
```

执行下面命令：

```
# g++ -c libB.cpp
# readelf -a libB.o
Symbol table '.symtab' contains 11 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     5: 0000000000000035     6 FUNC    LOCAL  DEFAULT    1 _ZL3foov
     9: 0000000000000000    22 FUNC    GLOBAL DEFAULT    1 _Z7subfuncii
    10: 0000000000000016    31 FUNC    GLOBAL DEFAULT    1 _Z5funcBii
```

可以看到foo是`LOCAL` ，funcB和subfunc都是`GLOBAL`。

根据这个特性，可以考虑把库中的符号变成`LOCAL`符号，只保留一个库中的符号是`GLOBAL`，这样链接的时候就不会报错符号重定义了。

使用objcopy命令的 `--localize-symbol` 选项可以把一个符号变成`LOCAL`。

```
--localize-symbol=symbolname
Make symbol symbolname local to the file, so that it is not visible externally. This option may be given more than once.
```

比如使用 `objcopy --localize-symbol=_Z7subfuncii` 命令。可以看到subfunc变成`LOCAL`了，funcB还是 `GLOBAL`。

```
# readelf -a libB.a | grep fun
     9: 0000000000000000    22 FUNC    LOCAL  DEFAULT    1 _Z7subfuncii
    10: 0000000000000016    31 FUNC    GLOBAL DEFAULT    1 _Z5funcBii
```

这个方法虽然可以消除链接时符号冲突问题，但有新的问题。如果冲突的符号很多，如何进行批量替换呢？可以使用`--localize-symbols`选项批量替换。

```
--localize-symbols=filename
Apply --localize-symbol option to each symbol listed in the file filename. filename is simply a flat file, with one symbol name per line. Line comments may be introduced by the hash character. This option may be given more than once.
```

`--localize-symbols`选项可以输入一个文件，文件中每行是一个符号名字。然而问题又来了，如何获取到所有冲突的符号名字？

一种方法是先编译，链接器打印出所有重定义符号错误后，使用grep, awk等工具提取出每个冲突的符号，导入到一个文件中。

```
libB.a(libB.o): In function `subfunc(int, int)':
libB.cpp:(.text+0x0): multiple definition of `subfunc(int, int)'
libA.a(libA.o):libA.cpp:(.text+0x0): first defined here
```

这个方法对C语言的符号比较适用，但是C++不能直接用，因为C++中的符号经过了编译器的name mangling，符号表中的名字不是代码和错误信息中的名字，比如`subfunc`在符号表中的名字是`_Z7subfuncii`，而传给localize-symbol选项的名字需要符号表中的名字。这就需要几次处理才能得到mangling之后的名字。

可以先用`readelf`命令导出.a中所有mangling之后的名字，然后用`c++filt`命令把mangling之后的名字进行demangle，得到代码中的变量名字（同时也是编译器报错的符号名字），然后和编译器报错的符号名字匹配一下，这样就可以把所有编译器报错的符号名字变成符号表中的名字，供`obj --localize-symbols`使用。



# objcopy --weaken

C++可重定位目标模块中的全局符号分为强符号和弱符号两种。强符号指的是函数和已经初始化的全局变量。弱符号指的是未初始化的全局变量。链接器使用如下规则来处理重定义的强弱符号：

```
1. 不允许多个同名的强符号
2. 有同名的一个强符号和多个弱符号，使用强符号的实现
3. 有同名的多个弱符号，选择任意一个弱符号的实现
```

根据这些信息，当有符号冲突时，肯定是多个强符号，这时可以把其中多个库的符号都变成弱符号，只保留一个强符号。

objcopy命令的weaken-symbol选项可以把一个库的某个符号变成弱符号。weaken-symbol选项说明如下：

```
--weaken-symbol=symbolname
Make symbol symbolname weak. This option may be given more than once.

--weaken-symbols=filename
Apply --weaken-symbol option to each symbol listed in the file filename. filename is simply a flat file, with one symbol name per line. Line comments may be introduced by the hash character. This option may be given more than once.

--weaken
Change all global symbols in the file to be weak. This can be useful when building an object which will be linked against other objects using the -R option to the linker. This option is only effective when using an object file format which supports weak symbols.
```

比如使用 ```objcopy --weaken libB.a``` 把libB.a中的符号全部变成弱符号。

执行前使用readelf命令可以看到subfunc是全局函数属于强符号。

``` 
# readelf -a libB.a | grep subfunc
     8: 0000000000000000    22 FUNC    GLOBAL DEFAULT    1 _Z7subfuncii
```

执行```objcopy --weaken libB.a```后：

```
# readelf -a libB.a | grep subfunc
     8: 0000000000000000    22 FUNC    WEAK   DEFAULT    1 _Z7subfuncii
```

subfunc已经变成了弱符号。

这个方法的缺点和allow-multiple-definition一样，如果不同库使用的是不同定义的符号，可能会出问题。



# objcopy --redefine-sym

以上方法都可以解决链接时的符号冲突问题，但都有隐患，可能造成使用了错误的符号定义。其实即使链接阶段没有报符号冲突错误，也可能会产生符号的错误使用。举个例子：

libA.cpp内容如下：

```
int subfunc_c(int a, int b) {
    return a + b;
}
int funcAA(int a, int b) {
    return subfunc_c(a, b);
}
```

libB.cpp内容如下：

```
int subfunc_c(int a, int b);
int funcBB(int a, int b) {
    return subfunc_c(a, b);
}
```

libC.cpp内容如下：

```
int subfunc_c(int a, int b) {
    return a - b;
}
```

main.cpp内容如下：

```
#include <cstdio>
int funcAA(int, int);
int funcBB(int, int);
int main() {
    printf("%d,", funcAA(2, 1));
    printf("%d\n", funcBB(2, 1));
    return 0;
}
```

libA.cpp和libC.cpp都定义了subfunc_c，libB.cpp引用了subfunc_c。我们把libB.o和libC.o打包成libB.a，libA.o打包成libA.a，然后编译链接，居然没有报错说subfunc_c冲突，而且执行输出的是 `3,3`，也就是说libB使用了libA的subfunc_c，而不是libB.a自己的subfunc_c定义。

```
# g++ -c libA.cpp
# g++ -c libB.cpp
# g++ -c libC.cpp
# ar rvs libA.a libA.o
# ar rvs libB.a libB.o libC.o
# g++ main.cpp libA.a libB.a
# ./a.out
3,3
```

解释这个问题，得从链接器的符号解析原理说起。

链接器从左往右扫描命令行传入的文件列表，包括.o和.a文件。扫描过程中，链接器维护三个集合，分别是：需要合并到可执行文件的符号集合E，未定义的符号集合U，已定义的符号集合D。

```
1. 初始E,U,D都为空。
2. 对于传入的每个文件，判断是.o文件还是.a文件。
如果是.o文件，扫描.o文件中的符号，全部添加到E集合中，根据每个符号是否定义，更新对应的未定义集合U和已定义集合D。扫描过程如果发现某个符号已经在已定义集合D中，报错符号重定义。
如果是.a文件，.a文件也是由一系列.o文件组成。链接器扫描每个.o文件，判断该.o文件中是否有未定义集合U中需要的符号，如果有，把该.o的符号全部添加到E集合中，更新未定义集合U和已定义集合D，如果没有，直接跳过该.o文件，不会把该.o文件合并到可执行文件中。
3. 全部输入文件扫描完成后，如果未定义集合U不是空的，报错有未定义符号。
```

根据这个原理，对于上面举的例子，libB.o和libC.o虽然都打包进libB.a，但是先扫描libA.a，subfunc_c已经在已定义符号集合D中了，后面扫描libB.a，扫描其中的libC.o，libC.o只有subfunc_c一个符号，而且发现subfunc_c已经在已定义集合D中了，这时libC.o就不会被合并到可执行文件中，也不会报错说符号重定义。

解决这个问题的一种方法是代码里重命名变量和函数，但这种方法比较难实现，尤其是一些第三方库可能连源代码都没有。另外一种方法是使用`objcopy --redefine-sym`重命名.a文件中的符号名字。

```
--redefine-sym old=new
Change the name of a symbol old, to new. This can be useful when one is trying link two things together for which you have no source, and there are name collisions.
--redefine-syms=filename
Apply --redefine-sym to each symbol pair "old new" listed in the file filename. filename is simply a flat file, with one symbol pair per line. Line comments may be introduced by the hash character. This option may be given more than once.
```

`--redefine-syms`可以传入文件批量替换符号名字，文件中每行是需要修改前的名字和修改后的名字，中间用空格分开。

这个方法可以彻底解决链接时的符号冲突问题，以及运行时多个符号定义导致使用了错误定义的问题。

当然使用这个方法也会遇到C++ name mangling的问题，解决方法参考上文`objcopy --localize-symbol`一节中的介绍。



# GCC visibility

前面介绍的都是静态链接库的处理方法，对于动态链接库(.so文件)，编译链接过程并不会报错，但是多个so文件定义了同一个符号，导致使用了错误定义的问题还是会出现。可以使用GCC visibility特性解决这个问题，具体参考 [GCC的符号可见性——解决多个库同名符号冲突问题](https://www.tuicool.com/articles/fy6Z3aQ) 一文。

