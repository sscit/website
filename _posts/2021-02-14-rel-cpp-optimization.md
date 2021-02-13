---
layout: post
title:  Optimizing REL Part 2 - Optimizing the C++ Code
date:   2021-02-13 18:05:31 +0100
---

In the last couple of weeks, I have been focusing on optimizing REL's C++ implementation, mainly for runtime performance and RAM consumption. In a series of blog posts, I will share my approach on how to measure performance metrics for C++ applications running on Linux, derive conclusions and optimize the code afterwards. This blog post will focus on C++ optimizations I applied to the code.

## Baseline and Test Data Set

In the [last blog post]({% post_url 2021-02-02-cpp-performance-measurements %}), I wrote about different tools and approaches, how to measure performance of C++ applications in Linux. I applied them to my implementation of REL, namely the core library and the surrounding CLI wrapper. To get representative values, I defined the [huge](https://github.com/sscit/rel/tree/main/test/huge) test data set, which consists of more than 400k requirements, distributed among 225 files, which sum up to around 500 MB. The amount of requirements is way more than even in large software projects, but the specification of this test project is rather simple. Nevertheless, enough food for the software, to spot the impact of different implementations in its measurements.

Baseline was [9fa79fd](https://github.com/sscit/rel/commit/9fa79fd58bc29eae549abc89e8b48fd4e01bd562) and the following values:

```
Wall Time: 5min 14s
RAM Consumption: 5,7 GB
```

Note that the wall time obviously depends a lot on the machine used to measure. As I am currently developing in an Ubuntu VM, the absolute value is not representative at all, but the change over time is for sure. Anyway, the baseline doesn't look like performant software, 5 minutes for 500 MB of data, and eating up more than 5GB of RAM. These values clearly reflect the state of the software at this point in time, I didn't apply any optimization, I rather focused on features and clean implementation.

## Optimizing the Lexer

After running `perf` on `rel_cli` and collecting first measurements, I quickly identified a couple of slow methods within class `Lexer`: They were called a lot during lexing of the files and consumed significant runtime, due to their poor implementation (see [05915cb](https://github.com/sscit/rel/commit/05915cb4982165e6e19dd2cbccdb7433422db01b)). Elimination of copies and using more tailored data structures already made an impact here. 

Additionally, I reworked the overall data flow of the library (see [1bdb698](https://github.com/sscit/rel/commit/1bdb698fb4c486a1c2471a8223a7f130a6358fad)). Before, all files were lexed first, and all tokens were stored in memory, which is not necessary at all. An improved workflow goes like this: Reading one file first, creating the tokens, and parsing them afterwards immediately. In this case, the tokens created can be deleted, before starting with the next file. This change alone had a significant impact on RAM consumption.

After the first iteration of optimizing the code, the new measurements looked like this:

```
Wall Time: 2min 47s
RAM Consumption: 2,7 GB
```

50% off! Sounds like a good start. There are always low-hanging fruits that can be grabbed easily.

## Eliminating Redundant Copying of Data

During the next iteration, I had a look at duplicate data storage within the key data classes `Token` and the ones used in the AST. For example, in all objects of class `Token`, I stored a string containing the full filename, where the token originated from. Same applied for objects of type `RdTypeInstance`, which represent a single type instance from the requirements data. For projects consisting of more than 400k requirements, 400k times the size of a string containing a filename indeed makes an impact. I replaced all those occurrences with a pointer to the filename. This optimization, and removing other redundant data, further decreased RAM consumption to less than 1 GB (see [921705d](https://github.com/sscit/rel/commit/921705df19a99372b9742245bfa8801092a65e61)). Impact on runtime was only minor though.

```
Wall Time: 2min 40s
RAM Consumption: 800 MB
```

## Using Multiple Threads to Read and Parse Requirements Data

While testing different configurations and collecting data, I noticed that `rel_cli` was running on a single core only. Of course, it was a single-threaded application, without any code running in parallel, therefore why should it benefit from all cores available in the system? Therefore I made some experiments to parallelize the library: A thread shall be created for every file, which lexes and parses the file and stores the resulting AST elements within the central data structure (see [b18f807](https://github.com/sscit/rel/commit/b18f8075686b7631d000d054d3dfd5f2d45c9d8c)). In C++17, spawning multiple threads is quite straightforward. I defined a function closure, which handles a single file. Additionally, to avoid race conditions, I introduced mutexes in `RdParser`, whenever shared data sources are updated. The results were quite whopping: Wall time went further down to nearly one minute only, another improvement by more than 60% compared to the last iteration. But of course, there is no free lunch: Memory consumption increased again significantly, of course, multiple threads in memory means that N files are processed in parallel and the resulting tokens have to be kept in RAM.

```
Wall Time: 1min 01s
RAM Consumption: 2,4 GB
```

In REL's CI validation on Github, that is fortunately based on way more powerful IT compared to my VM, the performance metrics can be seen, too. There the validation of the huge test project went down from approx. 1 minute runtime to less than [20s](https://github.com/sscit/rel/runs/1849966084). With this result, I consider requirement [perf1](https://github.com/sscit/rel/blob/main/requirements/5_performance.rd#L4) as fulfilled!

## Summary

To sum up, optimizing C++ code brings tremendous improvements in terms of resource consumptions. It is definitely worth it to spend this effort, because fast software is way more user friendly compared to slow execution times and is a real business value. 

Especially for a tool like REL, which is used by individual users as well as during CI validation, the business value is significant: For the user, optimized code for both RAM and runtime increases productivity and also enables users on "slow" machines, for example virtual machines, to use the tools without frustration. In corporations, where Windows is the dominant operating system, lots of people use Linux via virtual desktop or virtual machine, therefore this environment is a relevant use case. In CI, every second that is saved pays into the overall goal of fast iteration cycles and may helps to reduce cost for the cloud.

I am still playing around for further optimization. Due to the comprehensive test coverage of REL, introducing further changes imposes a very low risk to break something fundamental. I am still wondering if there is a sweet spot between the number of threads, CPU cores available and the corresponding RAM usage. And I would like to try out if mapping the whole file in memory first may introduce another speed gain. 