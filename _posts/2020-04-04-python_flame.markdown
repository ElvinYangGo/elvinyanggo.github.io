---
layout: post
title: "Python程序性能分析和火焰图"
author: "Yang"
header-style: text
tags:
  - Python
  - Flame Graphs
  - cProfile
  - flameprof
  - Performance
---

之前做过一个Python程序，用来解析Excel文件，经过一串复杂的处理，导出成其他不同格式的文件。随着需要处理的Excel文件越来越多，程序的执行时间也越来越长，需要对性能进行优化。

性能优化首先要找到瓶颈在什么地方，才能做针对性的优化。Python的性能剖析主要有下面几种方法：

- [ cProfile ](https://docs.python.org/2/library/profile.html)
- [line_profiler](https://github.com/rkern/line_profiler)
- [pyflame](https://github.com/uber-archive/pyflame)
- [pyinstrument](https://github.com/joerick/pyinstrument)

cProfile是Python标准库中内置的性能分析模块，非侵入式。cProfile生成的结果可以进一步导出成火焰图。

line_profiler主要做函数内每行语句的性能分析，需要侵入代码。如果已经知道哪个函数是瓶颈，需要对函数进一步分析，可以使用这个。

pyflame只能生成火焰图。

pyinstrument使用采样方法对函数的执行时间进行记录，开销比cProfile要小。没查到是否可以进一步生成火焰图。

我采用的是cProfile和火焰图。



#### cProfile

cProfile是Python标准库中内置的性能分析模块，C扩展，非侵入式，不需要修改代码。

使用方法：

`
python -m cProfile [-s sort_order] myscript.py
`

-s 指定输出的排序方法，可以传入tottime或者cumtime。tottime表示该函数本身的执行时间，不包括该函数调用的子函数。cumtime表示该函数累计执行时间，包括该函数调用的子函数。

cProfile输出如下：
```
         158683813 function calls (158373849 primitive calls) in 52.127 seconds

   Ordered by: cumulative time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.002    0.002   52.130   52.130 main.py:4(<module>)
        1    0.000    0.000   51.289   51.289 pipeline.py:19(execute)
        1    0.000    0.000   39.654   39.654 client_workbook_formatter.py:16(execute)
        1    0.000    0.000   39.654   39.654 client_workbook_formatter.py:31(format)
        4    0.000    0.000   39.653    9.913 client_workbook_formatter.py:48(format_message)
        4    0.003    0.001   33.730    8.433 client_workbook_formatter.py:72(format_first_column)
     1485    0.006    0.000   32.206    0.022 client_workbook_formatter.py:65(is_empty_row)
     1510    0.006    0.000   31.937    0.021 worksheet.py:406(iter_rows)
     1510    9.557    0.006   31.920    0.021 worksheet.py:363(max_column)
122321148   22.356    0.000   22.356    0.000 worksheet.py:371(<genexpr>)
       10    0.520    0.052    6.985    0.699 worksheet.py:748(move_range)
  2370654    1.733    0.000    6.465    0.000 worksheet.py:774(_move_cell)
        1    0.000    0.000    5.592    5.592 workbook_writer.py:12(execute)
        1    0.000    0.000    5.592    5.592 workbook_writer.py:24(write)
```

ncalls是每个函数被调用次数。

tottime表示该函数本身的执行时间，不包括该函数调用的子函数。

第一个percall表示tottime / ncalls。

cumtime表示该函数累计执行时间，包括该函数调用的子函数。

第二个percall表示cumtime / primitive calls，primitive calls 表示除去递归后本函数被调用次数。

filename:lineno(function)表示函数所在的文件名，行号和函数名。

cProfile是一种Deterministic Profiling。Deterministic Profiling是指记录所有函数每次的执行状况，而不是通过采样的方式来记录。通过采样的方式，性能开销会更小，但是记录可能会不够准确。[Python文档](https://docs.python.org/2/library/profile.html#what-is-deterministic-profiling)中说Python解释器内置了hook可以记录每个事件，反正解释器的开销已经很大了，所以相比起来使用Deterministic Profiling也增加不了多少开销，哈哈哈。

根据cProfile的输出再结合代码我们可以得出`client_workbook_formatter.py:65(is_empty_row)`是瓶颈所在。



#### 火焰图

虽然根据cProfile控制台输出和源码可以分析出瓶颈所在，但不够直观，下面介绍更直观的方式——火焰图。

首先使用 -o 选项生成cProfile的二进制性能结果。

`python -m cProfile [-o output_file] myscript.py
`

然后再用 [flameprof](https://github.com/baverman/flameprof) 生成火焰图。

`flameprof requests.prof > requests.svg`

最后用浏览器打开svg。
![](/img/in-post/post-flame-graph.png)

图分成上下两部分，上部的图是按照函数调用栈和执行时间排列。下部反方向的图按照函数执行时间比例从大到小排列。上部的图中execute是最顶层的函数，往上是它调用的子函数，直到调用链最底层的函数。宽度表示每个函数的执行时间占用的比例，越宽表示越耗时。

根据火焰图可以很直观地找到瓶颈。format_first_column调用了太多次is_empty_row，而is_empty_row的瓶颈在于item_rows。把iter_rows的结果在循环外事先算好，通过这个小改动，整个程序的执行时间从十几分钟减少到了100秒，可以满足目前的性能要求。
