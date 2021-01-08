---
layout: post
title:  Unit Testing with Google Test in Bazel
date:   2021-01-08 18:05:31 +0100
---

[Google Test]() is a well-establishe framework for unit tests in C++. It provides lots of features and can be used to write tests for own classes and their methods. Its integration in Bazel build system works quite well, with the benefit that it is not necessary to copy Google Test source files into the own repository, as they are downloaded on demand by Bazel during the build process. In this blog post, I will describe how [REL]({% post_url  2021-01-03-REL %}) uses Google Test. This approach can easily be transferred to every C++ development project that uses Bazel as build system.