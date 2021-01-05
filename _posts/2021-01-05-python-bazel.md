---
layout: post
title:  Building and testing C++ Python modules with Bazel
date:   2021-01-05 16:05:31 +0100
---

While working on [REL]({% post_url  2021-01-03-REL %}), I learned a lot about Python's integration in Bazel and the current limitations and workarounds of the build system.

As part of REL's Python integration, I created a `cc_library` called `rel_py` which includes the core REL C++ library and the necessary Python binding (using the ingenious [pybind11](https://github.com/pybind/pybind11) framework). If the library is built as dynamic library, the resulting _librel_py.so_ file can directly be imported in every Python script via `import` statement. It took me quite some time, though, to model the dependency between a `py_test` rule and the mentioned `cc_library`. My goal was to add an integration test to Bazel, which uses REL within Python, to read a toy model and test the basic functionality, like accessing all type instances, checking the API etc.

The integration test itself is a simple python script, that imports _librel_py.so_ and interacts with the API. I wrapped it into a `py_test` rule. At the moment, it is not possible, though, to model a dependency (`deps`) in Bazel from `py_test` towards `cc_library`, as Bazel only allows dependencies towards rules from the `py_` family. Therefore I tried the `data` - attribute, which allows specifying arbitrary dependencies, e.g. to test data. Unfortunately, with this approach, I was not able to specify the correct import paths for the Python runtime. During test execution, Python always complained, that the module that shall be imported cannot be found.

After searching on Stackoverflow and the Bazel bugtracker, I finally figured out the following approach, to get the dependencies right: Apparently it is necessary to define a dummy `py_library` first, which is modeled as dependency within the `py_test`. The `py_library` then uses the `data` attribute to point to a **`cc_binary`** rule, which is located in the **same folder** as the two py-rules, and is actually a copy of the original `cc_library`. The disadvantage of this solution is definitely, that the Bazel model is partially duplicated. Nevertheless, the obvious advantage, it now works and I can run an integration test via `bazel test`, that builds the Python binding library/binary of REL and runs a Python script, to test the functionality.

My solution in Bazel can be found here: [https://github.com/sscit/rel/blob/main/relpy/test/BUILD](https://github.com/sscit/rel/blob/main/relpy/test/BUILD)

Bazel Bugtracker issues related to this topic, that contain additional details:
- [https://github.com/bazelbuild/bazel/issues/1475](https://github.com/bazelbuild/bazel/issues/1475)
- [https://github.com/bazelbuild/bazel/issues/701](https://github.com/bazelbuild/bazel/issues/701)

