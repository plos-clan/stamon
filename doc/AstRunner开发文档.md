# AstRunner开发文档

> 注意：此文档旨在说明``AstRunner``（以下简称“虚拟机”）的工作原理

虚拟机的运行原理为：将二进制文件读取为AstIr，交给AstIrConverter类解析为Running-Ast，最后交给``vm/AstRunner.cpp``递归运行。

我们规定：Stamon编译后的二进制文件为**STVC文件**，文件后缀为``.stvc``

我们来逐步讲解。

首先是二进制文件读取为AstIr，这部分的实现位于``src/vm/AstIrReader.cpp``，AstIrReader类的主要接口有：

* ``AstIrReader(char* vmcode, int code_size, STMException* e)``：构造函数，vmcode为字节码在内存中的地址，code_size指字节码的大小，e为异常类
* ``bool ReadHeader()``：读取字节码头，读取失败则抛出异常并返回false，否则返回true
* ``ArrayList<AstIr> ReadIR()``：读取AstIr，返回由AstIr组成的ArrayList

想要完整的读取一个STVC文件，应该要先创建一个AstIrReader对象，然后先调用``ReadHeader``读取文件头信息，接着调用``ReadIR``来读取AstIr。调用这两个函数之后要分别检查是否有异常抛出。

接着是让AstIrConverter类解析为Running-Ast，这一部分在**写了Ast与AstIr之间的互转工具**部分里已经详细提及过了，故不再赘述。

最后是交给``vm/AstRunner.cpp``递归运行，``AstRunner``类采用了和语法分析器类似的结构，下面我们来看看重点的数据接口及接口：

AstRunner在递归执行Ast时的返回值为``RetStatus``类。RetStatus，全称*Return-Status（返回状态）*，用于指示当前代码运行状况，我们来看看RetStatus的定义：

```C++
class RetStatus {	//返回的状态（Return Status）
        //这个类用于运行时
    public:
        int status;	//状态码
        EasySmartPtr<Variable> retval;	//返回值（Return-Value），无返回值时为NULL
        RetStatus() {}
        RetStatus(const RetStatus& right) {
            status = right.status;
            retval = right.retval;
        }
        RetStatus(int status_code, EasySmartPtr<Variable> retvalue) {
            status = status_code;
            retval = retvalue;
        }
    };
```

其中的``int status``一行用于存储状态码，状态码有以下几类：

```C++
enum RET_STATUS_CODE {	//返回的状态码集合
    RetStatusErr = -1,	//错误退出（Error）
    RetStatusNor,		//正常退出（Normal）
    RetStatusCon,		//继续循环（Continue）
    RetStatusBrk,		//退出循环（Break）
    RetStatusRet		//函数返回（Return）
};
```

AstRunner的主要接口有：

1. 抛出运行时异常，这些异常分别是：

```C++
void ThrowTypeError(int type);
void ThrowPostfixError();
void ThrowIndexError();
void ThrowConstantsError();
void ThrowDivZeroError();
void ThrowBreakError();
void ThrowContinueError();
void ThrowArgumentsError(int form_args, int actual_args);
void ThrowReturnError();
void ThrowUnknownOperatorError();
void ThrowUnknownMemberError(int id);
```

2. 利用execute方法执行Running-Ast，它的函数原型是：

```C++
RetStatus execute(
    AstNode* main_node, bool isGC, int vm_mem_limit,
    ArrayList<DataType*> tableConst, ArrayList<String> args,
    STMException* e
);
```

虚拟机在执行过程中会向ObjectManager申请对象，来实现GC机制。

> ——摘自``工作日志/20240512.md``