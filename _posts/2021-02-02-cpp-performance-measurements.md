---
layout: post
title:  Optimizing REL Part 1 - Performance Measurements
date:   2021-02-03 18:05:31 +0100
---

In the last couple of weeks, I have been focusing on optimizing REL's C++ implementation, mainly for runtime performance and RAM consumption. In a series of blog posts, I will share my approach on how to measure performance metrics, derive conclusions and optimize the code afterwards. The first article talks about how to measure performance metrics in Linux, which tools are available and how to generate relevant data.

## Performance KPIs

Before optimizing an application, it is important to define performance metrics and collect data about current resource consumption. The data helps to identify hotspots, where lots of resources are consumed, and to compare the effect of different measures. Without measurements and data collection, all optimization efforts maybe address aspects that do not make an impact on the overall performance of the software, or make it even worse.

Useful metrics to collect are for example runtime and duration of the software while processing a realistic data set, or RAM consumption during the same scenario. Depending on your target system, size of binary or usage of network or other interfaces are maybe relevant KPIs as well. To optimize REL's C++ implementation, I have been focusing on runtime and RAM consumption. 

Before starting any measurements, it is also important to define a realistic data set. Measurements collected during "idle mode" of the software or in case of REL, with a tiny dataset, won't provide meaningful insights, as the execution is way too fast. Therefore before pursuing measurements, I defined [test data sets](https://github.com/sscit/rel/tree/main/test) of different sizes, to ensure that the system gets under load while processing the data.