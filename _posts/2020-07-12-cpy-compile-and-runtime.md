---
layout: post
title:  "Python 从源码到执行"
image: ''
date:   2020-07-12 12:30:00
tags:
- CPython
- Python
- Compiler
description: ''
categories:
- Python
---

## 0.介绍一下常见的编译模型: Java, Python, C
在今天的主题之前，先来了解下几个典型的编译模型。   
松本行弘先生，在讲解语言处理器构成时列举了一个通用架构。
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
处理器主要由三部分组成: 编译器(Compiler)，运行时(Runtime)，库(Lib)。   
* 编译器(Compiler): 顾名思义，就是编译源码的程序。通常情况下，它会将源码编译成运行时(Runtime)识别的中间码，但是在极端情况下，如 C 中，因为没有运行时(Runtime)，就直接输出机器码了。编译器(Compiler)在这过程中可能还会自己对源码进行优化，并剔除一些运行不必要的信息，比如注释等。
* 运行时程序(Runtime): 这个程序用于代码的具体执行，最被熟知的是 JVM，所以也可以把它叫做虚拟机好了
* 库(Lib): 库很好理解，就好比一个词典，运行这个程序所需要的一些额外支持。最基础的，标准库应该包含基本的 IO 库，如 `stdio.h`，还有平台所提供的系统调用等等。

Java 在这里就很有代表性，中规中矩的按照这个流程走。   
首先 Java 的编译器(Javac)会将源码 .java 的文件编译为字节码形式的 .class 文件。   
然后将文件中的字节码引入虚拟机中(JVM)，这里的 JVM 承担的就是运行时(Runtime)的任务。   
运行过程中从 JDK 中引入需要的库(Lib)。

那么 Python 会有什么不一样呢 ?   
Python 也是按照这个流程走的～～   
但是 Python 中编译器(Compiler)承担的工作比重相对较少，因为没有了复杂的语法检查还有类型校验等工作，大部分工作都在运行时完成，大部分错误也只有在运行过程中才能发现。   
像这种运行时(Runtime)部分承担大部分工作的语言，外观上给人一种像是直接从源代码执行的错觉，所以被叫做“解释型”。   
也是因为如此，Python 的执行效率远不及 Java ，还有 C 这些静态编译的语言。

最后 C 呢?   
C 又是另一个极端，C 语言的 GCC 编译器(compiler)异常强大，几乎包办了大部分工作。它能根据需要执行相应的编译优化，如转化机器不需要的变量名，加入混淆等等，几乎跳过了运行时(Runtime)直接输出包含机器码，输出文件经过链接器可以转换成平台可执行的文件。   
如果有了解过反编译的同学应该看过反编译回来的 C 源码可读性大打折扣，有时只能转到汇编了解程序的运行逻辑，相比较而言 Java 的 .class 文件中的字节码保留了更多信息，反编译回来的代码可读性更好一点。   
在我非常喜欢的美剧《硅谷》中也有这样一个桥段，Hooli 专门组织了一个团队反编译 Richard 的音乐程序，来获取他的数据压缩技术。   
这是一个非常有意思的过程，有很多书花了长篇大论专门介绍这个，这里不再展开了。

## 1.CPython 编译流程
我们先从编译器开始，Python 的编译器大致分为下面四步流程:

1. 将源代码解析为解析树(Parser Tree)
2. 将解析树转换为抽象语法树(Abstract Syntax Tree)
3. 将抽象语法树转换到控制流图(Control Flow Graph)
4. 根据流图将字节码(bytecode) 发送给虚拟机(ceval)

这是在最新的 CPython3.8.4 中的 python 源码编译及执行过程
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

    /* PyParser 包含了前述中的 step1 和 2 */
    mod = PyParser_ASTFromFileObject(fp, filename, NULL, start, 0, 0,
                                     flags, NULL, arena);
    if (closeit)
        fclose(fp);
    if (mod == NULL) {
        goto exit;
    }

    /* run_mod 包含 step3, 4 以及虚拟机的运作过程 */
    ret = run_mod(mod, filename, globals, locals, flags, arena);

exit:
    Py_XDECREF(filename);
    if (arena != NULL)
        PyArena_Free(arena);
    return ret;
}
```
大部分人对这一块应该没多大兴趣，我就简单带过，编译器主要会进行词法分析以及语法分析，遍历语法树生成流，最后生成虚拟机的机器码，也就是字节码。这里的每一点都包含很大的信息量，就不再详细叙述编译的细节了。下面解析一下 Python 的执行，那就要牵扯到虚拟机和字节码了。

## 2.虚拟机和字节码

虚拟机(vm): 这里所说的虚拟机不是 KVM，VMware 虚拟机。指的是在软件层面模拟了 CPU 执行逻辑的程序，其中最有名的应该是 JVM 了。在解释型语言 Python, Ruby 中同样包含了解析程序指令的虚拟机。 

字节码(bytecode): 字节码是相对于机器码的存在。机器码是 CPU 能读懂的机器指令，所有指令都包含在一个指令集里面，那字节码就是虚拟机能理解的指令。

那么在编程语言中，它们基于 CPU 的原理在软件层实现了一个指令集，相应的将程序翻译成虚拟机理解的字节码再加载到虚拟机中运行。   

这也给代码的移植性带来好处，只要相应的平台(Intel, ARM)上的操作系统(*nux, windows)安装有相应的虚拟机，就可以直接运行程序。

一个简单的例子:
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
上面的注释部分就是虚拟机要运行的指令部分，和在学校的时候学习的 x86 汇编非常相似，目前在 CPython3.8.4 中包含了163条指令(opcode)，不同版本之间指令有一些指令差异，在自己尝试的时候可能会看到不同的指令是正常的，感兴趣的话可以在源码(Include/opcode.h)中查看全部指令。   

另外需要提一点，CPython 中使用的是栈式虚拟机架构，相对的还有寄存器式虚拟机。

这是一个简单的打印 Hello World 的程序。   
我们来逐一解释以下:
* `LOAD_GLOBAL`: 将全局变量 print 压入(push)栈
* `LOAD_CONST`: 将常量 `Hello World`压入(push)栈
* `CALL_FUNCTION`: 执行 print 方法，弹出常量 'Hello World' 以及 print 变量，将结果压入(push)栈中。
* `POP_TOP`: 弹出(pop)栈顶元素，就是刚刚的 print 的返回值
* `LOAD_CONST`: 将常量 None 压入(push)栈
* `RETURN_VALUE`: 弹出(pop)栈顶元素作为最终返回值   

就是这样看起来复杂的六条指令拼凑出了 hello 函数。   

我觉得还有些疑问: CALL_FUNCTION 是如何 CALL 到这个函数的?  指令(opcode)旁边出现的数字有什么含义? 

我们得从字节码中找到答案。   

首先观察指令前面的数字 0,2,4,..,10，不难看出这是字节码的长度，0-2表示字节码的长度是 2 bytes，也就是说一条完整的代码指令其实是以字(word, 1 word= 2 bytes)的形式出现的，这里其实更适合叫做“字码”。   

然后是字码的构成，一个 word 有 16 个bit，前面说过 CPython 目前支持的指令(opcode)是163个，要怎么存呢?    

opcode 占用 8 bit，也就是说目前最多可以扩展到 256 个 opcode，另外 8 bit 存参数长度，这么说来一次函数调用最多只能压入(Push) 255 个参数(这个还真没试过，感兴趣可以试一下)。   

所以你看到的指令(opcode)后面的 0 和 1 其实是目前的参数长度，当 CALL 的时候，虚拟机会通过参数长度从栈顶向下检索调用位置运行函数。

这样差不多对字节码有了一定的认识了。

## 问题
通过上面逐一对指令进行了解析，相信大家对 python 又有了新了解了吧。但是还会有其他疑问。   
* print 这个全局变量是从哪里来的 ?   
* 'Hello Word' 字符串，还有 None 为什么应该是常量 ?   
* 这里只揭示了栈式虚拟机，那寄存器式虚拟机是什么样子的呢 ?   
* 另外一个老生长谈的问题，除了虚拟机负担了更多工作，还有哪些因素导致了 Python 的慢 ？   

这些问题一下也说不完，下次一定 :)

## 参考
* 松本行弘《编程语言的设计与实现》-> 1-2　语言处理器的结构
* Design of CPython's Compiler: https://devguide.python.org/compiler/

