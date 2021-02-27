---
layout: post
title:  Applying Bulk Changes to REL Model with Unix Tools
date:   2021-02-27 11:05:31 +0100
---

One of the design philosophies I was following during the development of the REL framework was to carefully evaluate every feature, whether it is really needed within the framework or if there are already solutions available that can be leveraged. In my professional life, I regularly observe that every software product can be literally destroyed by pressing more and more features into it, to cover every obscure use case ("we always did it like that") and at the same time sacrificing (code) quality and tests. This leads to unmaintainable software, which is bloated, slow and finally, when the original development team is not available anymore, will be replaced by something new. Focusing on a carefully selected set of features also means that other use cases have to be covered by alternative tools and approaches. The goal should always be to find the most efficient way to tackle a use case, by leveraging methods and tools that are already available and fit best. In today's blog post, I am writing about bulk changes that have to be applied to the whole requirements model. It's a use case every requirements engineer is very familiar with and I explain why I didn't implement dedicated support for this into the framework and rather rely on the use of Unix tools to reach the goal.

The REL framework does not contain any support to apply bulk changes to the requirements model. A bulk change is required, if for example a new attribute is introduced in the type definition. After introducing it, all existing type instances that are validated with the spec are not valid anymore, because they miss one attribute. Therefore it is necessary to modify all of them, and set a feasible value. Due to the fact that the whole model resides in text files within a repository, the change can easily be applied by using Unix tools like `find` and `sed`, which are able to search for patterns in the file, and execute basic operations. Additionally, it can be applied without any risk to destroy data or overload the system. Thanks to Git, a developer can modify its local copy and tweak it until the result is consistent again. Compare this with SQL bulk changes in the production database, which may lead to data loss and emergency roll backs of the database!

If `sed` is not powerful enough (is that really possible?), the next level would be to use REL's Python integration and write a small script that reads the REL model and prints it out again. That's more effort for sure, but in this case it's impossible that the data transformation cannot be resolved, as the script can perform any operation in between. The only aspect to consider in this case is that in order to read the requirements model with Python, it has to be consistent. Therefore the change within specification has to be kept "on hold", and an inconsistent model has to be printed out first, before the spec change is applied and miraculously, everything gets consistent again.

In the following paragraphs, I explain a couple of use cases, how to use Linux command line tools and apply bulk changes to a model. I use the [big](https://github.com/sscit/rel/tree/main/test/big) test model in REL's test folder, which contains around 4000 requirements and a decent specification, which allows me to play around with different use cases.

## Deleting an Attribute from Specification 

Let's assume the requirements team decided to delete one attribute from the [ComponentRequirement](https://github.com/sscit/rel/blob/main/test/big/spec.rs#L22) type definition, namely attribute [Is_implemented](https://github.com/sscit/rel/blob/main/test/big/spec.rs#L39). During the course of the project, the team learned that it doesn't make sense to have this information in the requirements model, because it can be derived out of the linked test cases and the test results.

Deleting it is trivial, just remove the [line](https://github.com/sscit/rel/blob/main/test/big/spec.rs#L39) out of the spec. But what about all the other 4000 requirements, that are now invalid, which also prevents committing the change to the server, because it fails during CI validation? Let's use the following Linux commands, to modify all files and delete the attribute:

```
/rel/test/big$ find . -name "*.rd" -type f -exec sed -i -e /Is_implemented/d {} \;
```

Firstly, `find` lists all files of type ".rd" and runs `sed` on them. `sed` simply removes every line containing "Is_implemented". To make it even more precise, space and colon that identify an attribute exactly can be included in the command. Checking the difference afterwards with `git diff` ensures that only the expected change has been created, wrong detections can be reverted. No matter how big the project is, this search and replace of the relevant attribute hardly takes a couple of minutes. When I compare this to database tools for requirements engineering, the advantages are obvious: If they do not support bulk changes, it's your (or the interns) job to replace all occurrences manually. If they do, it's often restricted to the admins, to prevent damage. If you are the lucky one with privileges to pursue bulk changes, it's often a delicate story to use the GUI and run the job: What if there is an error, and the whole production data gets a mess? Anyway, not comparable to patching text files locally and committing the delta afterwards without hassle.

## Adding a new Attribute to Type Definition

Let's have a look at another typical use case: Type definition [SystemRequirement](https://github.com/sscit/rel/blob/main/test/big/spec.rs#L3) is extended with a new attribute. After the owner, a new attribute called `Origin` shall be added, which is either `Internal` or `External`, depending on the requirement's origin. Setting all of them to `Internal` is simple:

```
/rel/test/big/SystemRequirements$ find . -name "*.rd" -type f -exec sed -i -e "/Owner :/a\\    Origin : Internal," {} \;
``` 

Some remarks on that:
- First of all, don't forget the leading whitespace, otherwise the text layout gets broken or another iteration with formatting script is required
- In this example, we benefit from the fact that all system requirements that shall be modified are located in a folder hierarchy. The tricky aspect is that component requirements also contain the `Owner` attribute, but shall not be modified. How to deal with it, if you cannot rely on the folder structure? Well, in this case, either the condition to look for within `sed`'s command line has to be more sophisticated, to detect the requirement's type via regex, or you use the approach sketched above with the Python script, that reads the whole model and writes it back again. While writing it back, you add your custom hooks that trigger only when you are at the right position to introduce the change.
- In the example, we always add `Internal` to the type instances, but what if it is necessary to evaluate a condition per system requirement to decide about that? My solution at the moment: Add the most likely value to all system requirements, and adapt the others manually afterwards. Depending on the numbers, this approach is often pretty pragmatic and works well. Otherwise, use the Python approach, reading everything and while writing it back, evaluate the condition.

## Using Python Script to modify Model

As I mentioned already, the most sophisticated approach is to write a Python script that reads the whole model and prints it back. While doing this, you can add your custom hooks to modify at the right position, by e.g. introducing a new attribute. I am thinking about writing the skeleton of such a script and adding it to the framework, which reads everything and writes it back the same way. In this case you can focus on writing the hooks, if you encounter the need to apply a bulk change to the whole model. Another benefit of such a script: It also acts as formatting tool, to remove unintended newlines, whitespaces, wrong indentation etc. After reading the model and writing it back, all files look consistent im terms of coding guidelines.