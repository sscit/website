---
layout: post
title:  Unit Testing with Google Test in Bazel
date:   2021-01-08 18:05:31 +0100
---

[Google Test](https://github.com/google/googletest) is a well-established framework for unit tests in C++. It provides lots of features and can be used to write tests for own classes and their methods. Its integration in Bazel build system works quite well, with the benefit that it is not necessary to copy Google Test source files into the own repository or use Git submodules, as Google Test's repository is downloaded on demand by Bazel during the build process. In this blog post, I will describe how [REL]({% post_url  2021-01-03-REL %}) uses Google Test. This approach can easily be transferred to every C++ development project that uses Bazel as build system.

First of all, Google Test has to be added as external dependency to the Bazel WORKSPACE file. Bazel's `new_git_repository` rule can be used for this. 

```
new_git_repository(
    name = "googletest",
    build_file = "gmock.BUILD",
    remote = "https://github.com/google/googletest",
    tag = "release-1.10.0",
)
```
[Source in REL project](https://github.com/sscit/rel/blob/main/WORKSPACE#L3)

In this snippet, Google Test's public repo on Github is referenced, and a dedicated tag is selected. If there is a new version of Google Test available, it is sufficient to update the tag in this rule, to change the dependency for all unit tests in the workspace.

As a second step, build file `gmock.BUILD` is created in folder `external`. It contains the Bazel definitions for Google Test, so that the referenced code is available as `cc_library`'s

```
cc_library(
    name = "gtest",
    srcs = [
        "googletest/src/gtest-all.cc",
        "googlemock/src/gmock-all.cc",
    ],
    hdrs = glob([
        "**/*.h",
        "googletest/src/*.cc",
        "googlemock/src/*.cc",
    ]),
    includes = [
        "googlemock",
        "googletest",
        "googletest/include",
        "googlemock/include",
    ],
    linkopts = ["-pthread"],
    visibility = ["//visibility:public"],
)

cc_library(
    name = "gtest_main",
    srcs = ["googlemock/src/gmock_main.cc"],
    linkopts = ["-pthread"],
    visibility = ["//visibility:public"],
    deps = [":gtest"],
)
```
[Source in REL project](https://github.com/sscit/rel/blob/main/external/gmock.BUILD)

The first library includes all Google Test related source files, whereas the second rule only contains the reference to Google Test's `main` function.

Now Google Test is available in the build environment as external dependency. If it is required as part of a build, it is downloaded and built (and cached) by Bazel. To use it for own unittests, `gtest_main` rule is referenced as dependency by the `cc_test` rule, which also includes the `*Test.cpp` - files, which contain the test cases.

```
cc_test(
    name = "RelLibUnitTest",
    srcs = glob(["**/*.cpp"]),
    deps = [
        "//rel-lib:rel_lib",
        "@googletest//:gtest_main",
    ],
)
```
[Source in REL project](https://github.com/sscit/rel/blob/main/rel-lib/test/unittest/BUILD)

`cc_test` is the "executable", which brings together the test cases (in this example, added to `srcs`), the actual source code that shall be tested (dependency to `rel_lib`) and the Google Test Framework, referenced as dependency, too.

To run it, `bazel test //path/to:RelLibUnitTest` is executed. Google Test logfiles are not printed out to the command line, they are stored in `/bazel-testlogs/path/to/RelLibUnitTest`.