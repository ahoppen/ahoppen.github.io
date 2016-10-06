---
layout: post
title: Using LLVM's testing tools lit and FileCheck for your own project
updated: 2016-10-06 17:00
---

For a university coursework I recently needed to develop a compiler that is able to compile a subset of C. I decided to use LLVM's tools `lit` and `FileCheck` for my testing infrastructure which are also used for testing clang and the Swift compiler.

This post will try to explain the basics of this testing infrastructure and explain how to set it up, so that the following code is a test case:

```
// RUN: %checkSwiftOutput

print("3 + 5 = \(3 + 5)") // CHECK: 3 + 5 = 8
if 6 * 7 == 42 {
    print("Hello world") // CHECK: Hello
}
```

### Introduction

Since the testing infrastructure was developed for testing a compiler, it often assumes that both input and output are available in text form and running a test means executing command on the command line. We will, however, see that `lit` on its own may also be used for other kinds of tests.

### `lit`

`lit` (abbreviation for **L**LVM **I**ntegrated **T**ester) is the test driver that powers LLVM based test suites. It is responsible for collecting all test files and running the tests cases included in them. It will do so by scanning a directory for files with certain extensions. In those, it will look for comments that contain the string `RUN:`. Each `RUN:` line specifies a terminal command that must return with exit code 0 in order for the test to succeed. 

For example, we may have a file `test.swift` in our home directory that looks like the following:

```
// RUN: swift ~/test.swift

print("Hello World")
```

`lit` will detect the first line as a test command and run `swift ~/test.swift`, compiling and running the test file. In this case `swift` is able to successfully compile and execute the test file and thus terminates with exit code 0. Hence the test succeeds. A shortcut that is often used in `lit` test files is `%s` which refers to the path of the current test file. So, one would most likely write the first line as:

```
// RUN: swift %s
```

Since `lit` only executes terminal commands which the user can write on its own, `lit` can be used as a test driver in basically all scenarios where input files have some kind of comment semantic that can contain the `RUN:` command. 

### `FileCheck`

`FileCheck` is a command line tool that is also developed as part of the LLVM project and can be used to assert that the output of a command matches a certain pattern. 

Again, this can be best explained with an example:

```
// RUN: swift %s | FileCheck %s

print("3 + 5 = \(3 + 5)") // CHECK: 3 + 5 = 8
if 6 * 7 == 42 {
	print("Hello world") // CHECK: Hello
}
```

In this example the test file will be executed and its output handed over to `FileCheck` which will check it against the pattern that is described in the same file. While `lit` looks for `RUN:` statements to execute commands, `FileCheck` searches the file for `CHECK:` statements. In this example `CHECK: 3 + 5 = 8` asserts that there exists a line in the output of `swift %s` that contains the string `3 + 5 = 8` and later a line that contains the text `Hello`. If all matches could be found `FileCheck` returns with exit code 0 and lets the test succeed. Note that position of the `CHECK:` statements does not matter and they could have also been placed at the beginning or end of the file.

In addition to the `CHECK:` directive `FileCheck` contains more directives which are described [here](http://llvm.org/docs/CommandGuide/FileCheck.html). Two that are often used are:

- `CHECK-NEXT:` asserts that the string is contained in the output line that follows the one that matched the last `CHECK:` statement. Using these statements, you can assert that blocks of text appear in the output.
- A series of `CHECK-DAG:` makes no assumptions about the order in which the lines appear. This is useful to, for example, assert that a list contains certain items but the order does not matter.

### Setting up `lit`

`lit` is a python project part of the main LLVM repo can be downloaded from the current master, e.g. on [GitHub](https://github.com/llvm-mirror/llvm/tree/master/utils/lit). 
Afterwards a `lit.cfg` config file needs to be created in the test directory that tells `lit` where to look for tests and how to execute them. A minimal setup is the following:

```
import lit

config.name = 'My Testsuite'

config.suffixes = ['.swift']

config.test_format = lit.formats.ShTest(execute_external=True)
```

It specifies the name of the testsuite for error messages, marks all files ending with `.swift` as test files and tells `lit` to execute the `RUN:` line as a shell script.

Now `lit` can be run using the following command:

```
/path/to/lit.py /path/to/test/dir
```

Pointing `lit` to a subdirectory will only execute the tests in that directory.

A full documentation of the options that can be configured in `lit.cfg` can be found in the official [documentation](http://llvm.org/docs/CommandGuide/lit.html#test-suites). I want to highlight some useful ones:

Just like `%s` is substituted to the current test file, custom substitutions can be specified. If we add

```
config.substitutions.append(('%checkSwiftOutput', "swift %s | FileCheck %s"))
```
to `lit.cfg`, we can use further simplify the first line in our test file to `// RUN: %checkSwiftOutput`.

Furthermore, if certain files should not be considered as tests, they can be excluded using `config.excludes = ['fileToExclude.swift', 'secondFileToExclude.swift']`.

### Setting up `FileCheck`

Unfortunately `FileCheck` cannot simply be downloaded but must be compiled with the entire `LLVM` project. Do do so on macOS, I suggest using [homebrew](http://brew.sh) and running `brew install llvm`. This will take some time (about 15 minutes), but afterwards `FileCheck-<version>` (`<version>` being something like `3.6`) can be found in `/usr/local/bin` and be used from the command line. I also created a symlink that points `FileCheck` to the concrete version for easiser use.

### Conclusion 
The LLVM testing architecture has clearly been created to do text based testing suited for compiler development and especially installing `FileCheck` needs some initial time. Afterwards, however, you get a pretty powerful, line-based output validator and easy to extend test suite.