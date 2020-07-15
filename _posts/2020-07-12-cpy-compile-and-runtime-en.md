---
layout: post
title:  "Python from source code to execution"
image: ''
date:   2020-07-16 00:05:00
tags:
- CPython
- Python
- Compiler
description: ''
categories:
- Python
---

## 0. Introduce the common compilation models: Java, Python, C
Before today's topic, let's take a look at typical compilation models.
Mr. Matz(Yukihiro Matsumoto) listed a general architecture when he explained the structure of language processor.

```
source code
    |
    |
   \./
----------                 ---------
|Compiler| ---mid code---> |Runtime|
----------                 ---------
                              /.\
                               |
                               |
                           ---------    
                           |  Lib  |
                           ---------    
```

The processor consists of three parts: compiler, runtime and lib.
* Compiler: As the name suggests. It's the program that compiles the source code. Typically, it compiles the source into mid code which will run in runtime. But sometimes, such as in C, these is no runtime, it will output machine code directly.In addition, the compiler may optimize the source and remove some unnecessary information, like comments and macro definitions.
* Runtime: This program is used for executing code. The most familiar one is JVM, so I think we can just call it vm.
* Lib: the library is easy to understand, like a dictionary, with some extra support needed to run the program. The most basic standard library shoud contain like IO libraries, such as `stdio.h`, ans system calls provieded by the platform.

Java is very representative here. It follows this process in a proper way.   
First, the java compiler (javac) compiles the source into a .class file contains bytecodes.   
Then put bytecodes in these files into JVM (vm), where the JVM undertaks the task of Runtime.

The required Library (Lib) is imported from JDK during the running process.

So what's different about Python?   
Python also follow this process~~   
But in Python, the compiler takes a relatively small proportion of the work, because thers is no complex syntax checking and type checking, most of the work is done at runtime. And most errors can only be found in the running process.   
Languages like this, where the runtime part is responsible for most of the work, look like they are executed directly from source code, so they are called interpretive language.   
Because of this, Python is far less efficient than Java and C, which are statically compiled languages.

What about C?
C is another extreme. The gcc compiler of C language is very powerful and almost does most of the work. It just outputs machine code which can be converted into the executable file of the platform through the linker.   
If you have any knowledge of decompilation, you should see that the readability of the C source code after decompiling is greatly reduced. Sometimes you must trans to assembly to understand the running logic of the program. In contrast, the bytecode in Java's .class file retains more information and decompiles back to the code is more readable.   
In my favorite TV series Silicon Valley, Hooli organized a team to decompile Richard's music program to obtain his data compression technology.   
It's a very interesting process. There are many books that have devoted a lot of time to this topic, which will not be expanded here.

## 1.CPython compilation process

Let's start with the compiler. The python compiler is roughly diivied into the following four steps:

1. Parse the source code into parser tree.
2. Transfrom parser tree into abstract syntax tree.
3. Transfrom ast to control flow graph.
4. Send the bytecode to the virtual machine according to the flow graph

This is the python source code compilation and execution process in the latest Cpython3.8.4:


```c
PyObject *
PyRun_FileExFlags(FILE *fp, const char *filename_str, int start, PyObject *globals,
                  PyObject *locals, int closeit, PyCompilerFlags *flags)
{
    PyObject *ret = NULL;
    mod_ty mod;
    PyArena *arena = NULL;
    PyObject *filename;

    filename = PyUnicode_DecodeFSDefault(filename_str);
    if (filename == NULL)
        goto exit;

    arena = PyArena_New();
    if (arena == NULL)
        goto exit;

    /* PyParser contains step1 and step2 */
    mod = PyParser_ASTFromFileObject(fp, filename, NULL, start, 0, 0,
                                     flags, NULL, arena);
    if (closeit)
        fclose(fp);
    if (mod == NULL) {
        goto exit;
    }

    /* run_mod contains step3 , step4 and runtime */
    ret = run_mod(mod, filename, globals, locals, flags, arena);

exit:
    Py_XDECREF(filename);
    if (arena != NULL)
        PyArena_Free(arena);
    return ret;
}
```
I don't want to go into too many compiler details here. Let's start runtime.

## 2.VM and bytecode

Virtual Machine(VM): the virtual machine mentioned here is not KVM or VMware vm. It refers to the simulation of CPU execution at the software layer, and the most famous one should be JVM. In the imterpreted language Python and Ruby also contains the virtual machine to execute the program.

Bytecode: Bytecode is very similar to machine code. Machine code is the machine instruction that the CPU can read. Then, the bytecode is the instruction executed by the virtual machine.

In programing languages, they create an instruction set in the software layer based on the principle of CPU, tanslate the program into bytecode and load into virtual machine to run.

A simple example:
```python
import dis

def hello(): 
    print("Hello World")

print(dis.dis(hello))

# 0 LOAD_GLOBAL              0 (print)
# 2 LOAD_CONST               1 ('Hello World')
# 4 CALL_FUNCTION            1
# 6 POP_TOP
# 8 LOAD_CONST               0 (None)
# 10 RETURN_VALUE

```

The comment above is the instruction part of the virtual machine. which is very samiler to the x86 assembly learned in school. In current CPython version 3.8.4, there are 163 instructions. They may have some differences between different CPython versions. When you try, you could see different instructions. Don't worry, it's right. If you are interesting in these instructions, you can view them in the source code (Include/opcode.h).

In addition, CPython is baesed on a stack virtual machine architecture. Another architecture is register virtual machine.

It's a simple program to print Hello World.   
Let's explain them one by one:
* `LOAD_GLOBAL`: push the global variable `print` into the stack
* `LOAD_CONST`: push the constant `Hello World` into the stack 
* `CALL_FUNCTION`: execute the `print` function, pop up the constant `Hello Word` and `print` variable, and push the result into the stack
* `POP_TOP`: pop the top element of the stack, which is the return value of the `print` function.
* `LOAD_CONST`: push the constant `None` into the stack
* `RETURN_VALUE`: pop the top element of the stack as the final return value

These six seemingly complex instructions make up the hello function.

I think there are still some questions: how dose CALL_FUNCTION find the function and call it? What is the meaning of numbers next to the instructions?

We have to find the answer in bytecode.

First of all, let's look at the numbers 0,2,4,..,10 in front of the instructions. It seems the length of bytecode. 0-2 means that the length of bytecode is 2 bytes. In other words, a complete bytecode is actually a word (1 word = 2 bytes).   
In fact, it is more suitable to be called "wordcode".

Then it is the composition of the wordcode. A word has 16 bits. As mentioned before, CPython currently supports 163 instructions. How can we save them?

Instructions takes 8 bits, so it can be expanded to 256 instructions at present, and the other 8 bits save the length of arguments.   
In this way, a function can only push 256 parameters (I haven't tied before, but you can if you are interested).

So the 0 and 1 after the instructions you see are actually the current parameter length. When calling, the virtual machine will retrieve the calling position from the top of stack through the parameter length to call it.

This almost has a certain understanding of bytecode

## Reference
* まつもとゆきひろ<まつもとゆきひろの 言語のしくみ> 1-2 language processor
* Design of CPython's Compiler: https://devguide.python.org/compiler/

