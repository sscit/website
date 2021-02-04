---
layout: post
title:  Optimizing REL Part 1 - Performance Measurements
date:   2021-02-03 18:05:31 +0100
---

In the last couple of weeks, I have been focusing on optimizing REL's C++ implementation, mainly for runtime performance and RAM consumption. In a series of blog posts, I will share my approach on how to measure performance metrics, derive conclusions and optimize the code afterwards. The first article talks about how to measure performance metrics in Linux, which tools are available and how to generate relevant data.

## Performance KPIs

Before optimizing an application, it is important to define performance metrics and collect data about current resource consumption. The data helps to identify hotspots, where lots of resources are consumed, and to compare the effect of different measures. Without measurements and data collection, all optimization efforts maybe address aspects that do not make an impact on the overall performance of the software, or make it even worse.

Useful metrics to collect are for example runtime of the software while processing a realistic data set, or RAM consumption during the same scenario. Depending on your target system, size of the binary or usage of network or other interfaces are maybe relevant KPIs as well. To optimize REL's C++ implementation, I have been focusing on runtime and RAM consumption. 

Before starting any measurements, it is also important to define a realistic data set. Measurements collected during "idle mode" of the software or in case of REL, with a tiny dataset, won't provide meaningful insights. Therefore before pursuing measurements, I defined [test data sets](https://github.com/sscit/rel/tree/main/test) of different sizes, to ensure that the system gets under load while processing the data.

## Measuring Runtime

Linux Tool `time` provides an easy way to measure the runtime of a program. Just run it with the program and its parameters:

```
time bazel-bin/rel-cli/rel_cli -r ./test/big 
```

It then returns three values, namely 

```
real	0m2.509s
user	0m2.461s
sys	    0m0.032s
```

`real` describes the time elapsed from start to finish of the program, the so called wall clock time. It includes waiting times and time, where the process was not scheduled at all.

`user` is the time that was actually spent within the user space process, and `sys` shows the time spent in the kernel within the process.

If `time` is run multiple times and with different software versions, a data set can be built up and the effect of changes can be evaluated.

### Identify Expensive Code

`time` gives a first indication about the overall runtime, but it does not provide any details about which parts of the code take a long time. To get this kind of information, `perf` can be used. It executes the process, similar to `time`, but collects data during execution, which can be used to generate a report afterwards.

```
sudo perf record <binary> <binary_options>
sudo perf report
```

The report lists functions sorted by their CPU consumption during the execution.

```
Samples: 2K of event 'cpu-clock:pppH', Event count (approx.): 740750000
Overhead  Command  Shared Object        Symbol
  15.69%  rel_cli  libc-2.31.so         [.] __memcmp_avx2_movbe
  12.55%  rel_cli  rel_cli              [.] std::_Rb_tree<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<
   8.64%  rel_cli  rel_cli              [.] Lexer::Read
   4.86%  rel_cli  libstdc++.so.6.0.28  [.] std::istream::get
   3.27%  rel_cli  rel_cli              [.] Lexer::IsOperatorOrKeyword
   3.24%  rel_cli  libstdc++.so.6.0.28  [.] std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >::compa
   3.17%  rel_cli  libstdc++.so.6.0.28  [.] std::istream::sentry::sentry
   3.10%  rel_cli  libstdc++.so.6.0.28  [.] std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >::_M_re
   2.73%  rel_cli  libc-2.31.so         [.] __memmove_avx_unaligned_erms
   2.40%  rel_cli  libc-2.31.so         [.] __memmove_avx_unaligned
   2.19%  rel_cli  libc-2.31.so         [.] malloc
   2.16%  rel_cli  libc-2.31.so         [.] _int_malloc
   2.13%  rel_cli  rel_cli              [.] FileReader::GetChar
   1.89%  rel_cli  rel_cli              [.] Lexer::IsOperator
   1.86%  rel_cli  libstdc++.so.6.0.28  [.] std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >::_M_re
   1.65%  rel_cli  libc-2.31.so         [.] __strlen_avx2
   1.52%  rel_cli  rel_cli              [.] Lexer::IsLinebreak
   1.38%  rel_cli  libstdc++.so.6.0.28  [.] operator new
   1.21%  rel_cli  rel_cli              [.] Lexer::CheckStringandAddToken
   1.18%  rel_cli  rel_cli              [.] SlidingWindow::pop_front
   1.15%  rel_cli  libc-2.31.so         [.] _int_free
   1.15%  rel_cli  libstdc++.so.6.0.28  [.] std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >::_M_ap
   1.15%  rel_cli  rel_cli              [.] Lexer::AddTokenToList
   1.15%  rel_cli  rel_cli              [.] RdParser::ReadString
   1.11%  rel_cli  libc-2.31.so         [.] cfree@GLIBC_2.2.5
   1.11%  rel_cli  libc-2.31.so         [.] isalnum
```

In this excerpt, we can see that `rel_cli` application spends most of its time on memcopy operations, due to building up AST and token lists containing lots of strings. Additionally, method `Read()` within class `Lexer` is worth investigating. A report like this is very useful to focus on the most expensive methods first, that may offer most potential for optimizations, too. `perf` tool is pretty straightforward to use, too, and doesn't require special compilation flags. The release binary including optimized code can be used for analysis.

## Measuring RAM consumption

Another resource that is worth optimizing is the amount of RAM required during execution. `/bin/time/ -v` provides valuable insights on this KPI:

```
/bin/time -v bazel-bin/rel-cli/rel_cli -r ./test/big 
```

Besides other information, it returns the `Maximum resident set size (kbytes):`, which indicates the maximum amount of physical RAM allocated by the process during runtime. By running `/bin/time/` multiple times with different software versions, the impact of optimization measures in terms of RAM consumption can be evaluated.

In the next blog post, I will describe how I evaluated REL's performance with these tools, and describe measures that improved the KPIs over time.