---
layout: post
title:  "Writing an LLVM Pass"
date:   2025-08-09 13:00:00 -0300
category: LLVM
lang: en
---

### Introduction

After a long time since the [last]({{ "/compilers/2024/11/03/intermediate-representation.html" | relative_url }}) post (and only post at my blog), it's time to write something new in here. Today, we're going to address a more practical topic.

In this post, I'm gonna explain (or at least _try_ to) how to implement a pass for the LLVM infrastructure. I know that there are many great tutorials available online, but that's not the case for Portuguese only readers. Since I already written the Portuguese version of this post, I figured it wouldn't be that big of an effort to just translate it here.

This post is written for those who have little to no knowledge on the compilers field, but are interested on it. However, it's expected that the reader is familiar with C++ to understand the implementations covered in this post.

Here's a break down of what you'll see here:
- The LLVM Infrastructure;
- What are LLVM passes;
- How to write an LLVM pass;
- How to compile and run a pass.

Disclaimer: I've tried my best to ensure that the information written here is correct, but the [official LLVM documentation](https://llvm.org/docs/WritingAnLLVMNewPMPass.html) is the definitive source of information and best practices.

### LLVM

The LLVM Infrastructure is a toolkit that allows one to build (and use) a lot of compilers. With it, one can quickly make a compiler for any programming language that generates a very efficient machine code.

Some tools that we can highlight are:
- **LLVM Core:** allows a lot of optimizations on the source code, and also generates machine code for most of the CPU architectures out there;
- **clang:** a native LLVM C/C++ compiler;
- **libc and libc++:** a very optimized implementation of the C/C++ standard library.

Of course, there are many other tools, and you can check then on the [LLVM website](llvm.org).

But what matters the most for us now is the LLVM Intermediate Representation (LLVM IR). It stays in the between of programming language and assembly, and is a great representation for running analysis and optimizations. When you engineer a compiler for some language using the LLVM, usually the process consists of creating a compiler front end for translating the source code into LLVM IR, and then use the LLVM tools for doing some magic on the program and generate a very optimized machine code for some target architecture.

### LLVM Passes

The LLVM Infrastructure is very modular. Part of using this modularization is done through the LLVM passes. They are individual plugins that performs some tasks on the code being compiled.

To run an LLVM pass, you need to run the `opt` tool, that is part of the LLVM tool belt. A pass receives a program as input and "pass" through it, collecting information or modifying it.

Because of this, a pass can fit into 3 different categories:

- Analysis: passes that extract information about programs and make it available to be used by other passes;
- Transformation: passes that modify a program in some way;
- Utility: helper passes that don't fit neither in the analysis nor in the transformation categories, but helps visualizing information.

Passes can operate on different abstraction levels, but usually we work on 2 levels:

- Modules: roughly speaking, represents single source files of the program. They contain functions, global variables and metadata about these source files;
- Functions: represent a function of a program, containing the function signature (name, parameters, return type), basic blocks, attributes and metadata.

Based on these abstraction levels, we have two types of passes: function passes (when the pass run on the functions of a program one by one) and module passes (when the pass run on the modules one by one).

Lastly, a quick note on history: LLVM has two pass management systems. The old one is called Legacy Pass Manager (which is now deprecated), and the modern one is called New Pass Manager (NPM). All my posts about LLVM passes, including this one, will always refer to the NPM.

### Developing an LLVM Pass

Now that we know what passes are, let's talk about how to implement one. First, there are two different approaches for developing LLVM passes: "inside the tree" and "out of the tree".

I must confess that I've never written a pass "inside the tree", but the idea is that you place your pass implementation inside the LLVM source code folder, together with the other passes. To compile it, you have to recompile the `opt` tool.

Instead, let's focus on the "out of the tree" method, which I'm familiar with (and, from my point of view, it's far more elegant). The beautiful thing about this method is that you can implement your pass in any folder you want on your system. You just have to add some boilerplate code to make it work.

The structure of our pass will have a simple file organization that is common throughout the out of the tree passes:

```
my_pass/
    |
    +--include/
    |   |
    |   +--MyPass.h
    |
    +--lib/
    |   |
    |   +--MyPass.cpp
    |   +--MyPassPlugin.cpp
    |
    +--CMakeLists.txt
```

For writing the pass, we will use C++ for the implementation and CMake for managing the build process. If you don't know how to use CMake, no worries. I'll show you a standard setup for using it for compiling LLVM passes.

In this structure, we have:
- `MyPass.h`: our header file;
- `MyPass.cpp`: our implementation file; and
- `MyPassPlugin.cpp`: where we register our pass within the LLVM Plugin system, so that the `opt` tool can find and run it.

For our first pass, let's implement a simple analysis pass: for each function, we'll print the name of the function and the number of basic blocks that it has. Notice that, for this purpose, we can develop a function pass, because we can analyze each function independently. 

I'll explain now how to implement each file, starting by:

#### MyPass.h

The code of the header consists in a declaration of the class that defines the pass (this is an LLVM pass, a C++ class):

```cpp
#ifndef MY_PASS_H
#define MY_PASS_H

#include "llvm/IR/PassManager.h"

namespace llvm {

class MyPass : public PassInfoMixin<MyPass> {
public:
    PreservedAnalyses run(Function &F, FunctionAnalysisManager &FAM);
};

}

#endif // MY_PASS_H
```

Let's dive deeper into the details.

The class `MyPass` inherits from the class `PassInfoMixin<MyPass>`, which is an example of the [CRTP](https://en.cppreference.com/w/cpp/language/crtp.html) (Curiously Recurrent Template Pattern) pattern. It looks strange at first, but it's a trick that lets LLVM's mix-in class automatically configure a lot of boilerplate information needed for integrating our pass into the system.

The class have one function declaration: `run`, that is responsible for running the pass (it's almost equivalent to a `main` function of a program).

This function have the following signature:
- It takes two parameters:
    1. `Function &F`: A reference to the LLVM IR function the pass is currently analyzing.
    2. `FunctionAnalysisManager &FAM`: A reference to an object that manages the execution of different analysis for the `Function` type.
- It returns a `PreservedAnalyses` object, that will be explained further.

Another important thing to notice is that we are declaring this class inside the `namespace llvm`. That's because the pass must be declared inside the LLVM namespace to be properly integrated (becoming `llvm::MyPass` when seen from outside).

Now we'll see how we implement the function `run`.

#### MyPass.cpp

Remember what we want to do with our pass. For each function, we want:
- The name of the function; and
- The number of basic blocks inside it.

Luckily, all this information can be easily retrieved, so the code becomes very simple:

```cpp
#include "MyPass.h"

using namespace llvm;

PreservedAnalyses MyPass::run(Function &F,
                                FunctionAnalysisManager &FAM) {
    outs() << F.getName() << " " << F.size() << "\n";
    return PreservedAnalyses::all();
}
```

In this function, we are using the function `outs()`, LLVM's output stream function. This is similar to the `std::cout` function from the `iostream` library, but optimized for LLVM types, and that can handle the `StringRef` type, which is returned by the `getName()` function.

The `getName()` method from the class `Function` returns the name of the function being analyzed, while the `size()` method returns the number of basic blocks in that function.

Lastly, we return `PreservedAnalysis::all()`, so let's finally explain what is this `PreservedAnalysis` type. It represents a set of analysis that we are guaranteeing that our pass preserves. In this context, the function `all()` says that our pass guarantee that **every** analysis is preserved. On an analysis pass, this should be always true, but when we deal with transformation passes, then there is a chance that not all the analyses are preserved. 

#### MyPassPlugin.cpp

Now that our pass is implemented, we need to register it within LLVM, in order to be able to run it. For this, we'll write a bunch of boilerplate code to connect our `MyPass` class to the `opt` tool.

First, the necessary headers:

```cpp
#include "MyPass.h"
#include "llvm/Passes/PassBuilder.h"
#include "llvm/Passes/PassPlugin.h"

using namespace llvm;
```

Where we don't have anything special to comment. Next, the function that register the pass pipeline:

```cpp
bool registerPipeline(StringRef Name, FunctionPassManager &FPM,
                      ArrayRef<PassBuilder::PipelineElement>) {
    if (Name == "my-pass") {
        FPM.addPass(MyPass());
        return true;
    }
    return false;
}
```

Here, we're saying that, when `opt` is asked to run the pass `my-pass`, a pipeline composed by `MyPass()` will be registered. Notice that `MyPass()` is the builder function of the class `MyPass` that we've defined previously. The return value is saying whether the pass with name `my-pass` was found or not. Also, you could add multiple passes here, creating a custom pipeline, where the order in which you add the passes is the order that they are gonna be executed in that pipeline.

```cpp
PassPluginLibraryInfo getMyPass() {
    return {
        LLVM_PLUGIN_API_VERSION, "my-pass",
        LLVM_VERSION_STRING, [](PassBuilder &PB) {
            PB.registerPipelineParsingCallback(registerPipeline);
        }
    };
}
```

Now we are defining a function that says how the pass must loaded. The type `PassPluginLibraryInfo` is a struct that keeps the LLVM Plugin API version, the class name, the LLVM version and a function that registers the pass pipeline (in this case, the function we implemented above).

Finally, we indicate how to initialize the plugin (that says how the pass is loaded) with:

```cpp
extern "C" LLVM_ATTRIBUTE_WEAK PassPluginLibraryInfo
llvmGetPassPluginInfo() {
    return getMyPass();
}
```

With this code completed, let's see how we are gonna compile everything.

#### CMakeLists.txt

In order to build our pass, let's make use of CMake for managing the process for us. I suppose that you have LLVM installed and, more specifically, compiled and installed it using CMake.

If you don't, I recommend following the official [tutorial](https://llvm.org/docs/CMake.html) from LLVM.

Let's start the CMake file with two mandatory lines, where we define the minimum version of cmake for compiling the project and the name of the project:

```cmake
cmake_minimum_required(VERSION 3.20)
project(MyCoolPass)
```

Then we have some boilerplate to find the LLVM libraries and to configure some compilation flags. This is standard for any out of the tree pass:

```cmake
set(CMAKE_CXX_STANDARD 17 CACHE STRING "")

set(LLVM_INSTALL_DIR "" CACHE PATH "LLVM installation directory")
set(LLVM_CMAKE_CONFIG_DIR "" "${LLVM_INSTALL_DIR}/lib/cmake/llvm/")
list(APPEND CMAKE_PREFIX_PATH "${LLVM_CMAKE_CONFIG_DIR}")

find_package(LLVM REQUIRED CONFIG)

include_directories(${LLVM_INCLUDE_DIRS})

if(NOT LLVM_ENABLE_RTTI)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-rtti")
endif()

set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/lib")
```

Now, let's define how CMake should compile our pass. We create a library `MyPass` with type `MODULE` from our source files:

```cmake
add_library(MyPass MODULE
    lib/MyPass.cpp
    lib/MyPassPlugin.cpp)
```

Lastly, we say to the compiler where our library `MyPass` should search for header files:

```cmake
target_include_directories(MyPass PRIVATE
    "${CMAKE_CURRENT_SOURCE_DIR}/include")
```

In this way, we are allowing that the files `MyPass.cpp` and `MyPassPlugin.cpp` to "see" the header `MyPass.h` without having to write the exact path for it (relatively, `../include/MyPass.h`). Remember that, when including this header file, we did it with `#include "MyPass.h"`.

Now, we have an implemented pass, with instructions on how to register it on the pass pipeline of LLVM, and a CMake file describing how to compile it. Now it's time to test and see if everything is working.

### Testing a Pass

Let's start by compiling the pass. First, we'll use CMake for generating the build files and Unix Makefiles to compile them:

```bash
mkdir build
cd build
cmake ..
make
```

If everything is right (and you didn't change the `CMAKE_LIBRARY_OUTPUT_DIRECTORY` variable), your compiled pass should be located at `build/lib/libMyPass.so`.

Now we can run it, but first, we need some code for the pass to run. Remember that we made a pass that analyzes... **code**. Let's use the following code (that I'm lazily naming it `a.c`), with a Fibonacci recursive function:

```c
#include <stdio.h>

int f(int x) {
    if (x < 2) return x;
    return f(x-1)+f(x-2);
}

int main() {
    printf("%d\n", f(5));
    return 0;
}
```

Then we compile it to LLVM IR for allowing our pass to understand it. In order to do so, we use clang with some parameters:

```bash
clang a.c -Xclang -disable-O0-optnone -S -emit-llvm -o a.ll
```

The parameters `-Xclang -disable-O0-optnone` prevent LLVM from marking the functions of this code as non-optimizable, which would block our pass from running on them. The parameters `-S -emit-llvm` makes clang spill out the code in LLVM IR instead of producing a binary code. The generated code should look like:

```llvm
...
define dso_local i32 @f(i32 noundef %0) #0 {
  ...
  br i1 %5, label %6, label %8

6:  ; preds = %1
  ...
  br label %16

8:  ; preds = %1
  ...
  br label %16

16: ; preds = %8, %6
  %17 = load i32, ptr %2, align 4
  ret i32 %17
}

define dso_local i32 @main() #0 {
  ...
  ret i32 0
}

...
```

Notice that function `@f` has 4 basic blocks (0, whose name is omitted, 6, 8 and 16) and function `@main` has only 1 basic block (0, whose name is omitted).

Finally, we can run our pass with the following command:

```bash
opt -disable-output -load-pass-plugin lib/libMyPass.so -passes="my-pass" a.ll
```

Note that since our pass isn't included in the set of default passes of LLVM, we need to load our compiled pass with `-load-pass-plugin`. Also, `-disable-output` is saying to `opt` to not output our transformed IR file as a binary code. The output stream of the pass is not affected by this flag, as counter intuitive as it might sound.

After executing, the expected output is:

```
f 4
main 1
```

Our pass have successfully analyzed the IR, which means it worked!

### Conclusion

Now you have your own LLVM pass. This is a very simple analysis pass, indeed, but the idea here was to explain the concepts of an LLVM pass and how to develop them, showing how to declare then, implement its function, link it into the LLVM and configure cmake for compiling it.

But this is just the beginning. In the next post about LLVM pass implementation, we'll work on a code transformation pass that will be more useful (Spoiler: we are gonna count how many times each edge in the CFG is traversed).

If you have any questions about this post, remember that my email in on the [about]({{ "/about" | relative_url }}) page.

### References

- https://llvm.org/
- https://llvm.org/doxygen/