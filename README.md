# DPI-C

translate the dpic part of IEEE std 1800-2017 (System Verilog)


[正文](https://github.com/zhuzhzh/dpic/blob/master/dpic.md)

# 35. Direct Programming Interface

## 需求
随着时代的发展，现在的芯片规模越来越大，哪怕模块级的验证环境也需要相当长的build时间，各种仿真工具也在改进编译和运行性能，还发明了增量编译。但无论如何turnaround的时间还是比较长，而且方法越复杂越容易出错。而DPI-C则比较简单，能够解决某些场景下的问题。

## 适用范围
DPI-C比较适用于SV和外部语言间的“简单数据“交互

## 翻译约定
subroutine == 子例程 == task and function
task == 子任务
function == 子函数
foreign language == 外部编程语言
在某些情况下会保留英文表述不做翻译


## 通用

下面章节主要包括：
- DPI 子任务和子函数
- DPI层
- 导入和导出子函数
- 导入和导出子任务
- 禁用 DPI 子任务和子函数

## 35.2 总述

本章节主要描述了DPI以及接口中SV层的细节。
DPI是SV和外部编程语言的接口。它包含两个独立的层次： SV层 和 外部编程语言层。两侧是被完全隔离的，也就是说它们彼此是透明的， 外部编译语言和SV侧完全无关。无论是SV编译器还是外部编程语言编译器都不会分析对方的代码。不同编程语言应该都可以被一样的SV层支持。但目前SV只定义了对C的支持。
这个接口的动机是双面的。方法学上的要求是整个系统可以由多种语言组成而不仅仅是SV，另一方面是已有的代码（比如C,C++)能够被很容易地集成进来。

DPI遵循black-box原则。描述和实现隔离。真正的实现方法对于系统的其他部分是透明的。 因为外部语言对SV来讲也是透明的，不过这个标准目前只定义了C链接语义。 SV和C通过子函数为基本单元来封装和交互。所以任何子函数都可以被当成黑盒子，它的实现或者在SV中，或者在外部编程语言中，它的实现修改了但无需修改调用侧。

### 35.2.1 tasks/functions

DPI允许语言之间的相互子函数调用。展开来讲就是，SV可以调用外部编程语言里定义和实现的子函数，对SV来讲就是**imported functions**; 外部编程语言可以调用SV里定义和实现的子函数，对SV来讲就是**exported functions**，需要在SV中作导出声明。DPI允许通过子函数的参数和结果来在两个域中传递数据。这个接口没有固有的开销。

外部语言代码也可调用SV task, 原生的SV代码也可能调用imported tasks.
imported tasks和原生的SV task在语义上是一样的：
- 无返回值
- 可以消耗仿真时间

所有DPI 子函数都是不消耗仿真时间的，DPI无法提供除数据交换和控制权转移以外的同步手段.

每个imported的subroutine都必须先声明。
import声明可以写在任何SV subroutine允许的地方。同一个外部subroutine可以被多处调用。
imported subroutine可以有0个可多个 input, output, inout 参数。
imported tasks只能返回void value; imported function可以返回值或void value.

### 35.2.2 Data types

SV 数据类型必须是能够跨越SV和外部语言的类型, 称为富类型：
- void, byte, shortint, int, longint, real, shortreal, chandle, string
- scalar value of bit and logic
- packed arrays, structs, uniions composed of types bit and logic
- emumeration types (interpreted as the associated real type)
- types constructed from the following constructs:
    - struct
    - union(packed only)
    - unpacked array
    - typedef

下面是一些注意点：
- enum 不能被直接支持，实际上会被解释为enum type关联的类型
- 在exported DPI subroutine里， 声明形参为dynamic array容易出错
- SV data的实际内存结构对于SV是透明的。
Function的返回值必须是small values:
- void, byte, shortint, int, longint, real, shortreal, chandle, string
- scalar value of bit and logic
而imported function的形参可以是open arrays

#### 35.2.2.1 内存数据结构
DPI并没有对于数据在内存里的存在形式做任何约束， 所以这是平台相关的，和SV无关

## 35.3 DPI的两层

DPI由独立的两层组成： SV层和外部编译语言层。他们不相互依赖。虽然SV可以支持各种编译语言，但目前SV标准只对C做了详细定义。不同的外部编程语言可以要求SV实现使用适合的函数调用协议，参数传递，链接机制。但这对于用户来讲是透明的。SV标准只要求仿真器的实现者支持C协议和链接。

### 35.3.1 SV layer

不依赖于外面到底是什么语言。外部语言的实际函数调用规则和参数传递机制对于SV来讲是透明和无关的。SV同等看待所以外部语言接口。SV侧的接口语义和外部编译语言侧的接口无关。

### 35.3.2 foreign language layer
定义实参是如何被传递的， SV data（如logic, packed）是如何表示的。如何与类C的类型转换
对于不同的外部语言，SV端的代码应该是一样的

## 35.4 imported and exported functions的全局命名空间

每个imported subroutine最终最会有一个全局符号标志；而每个exported subroutine会定义一个全局符号标志。
它们必须唯一。它们遵循C的命名规则，必须以字母或_开头。
import和export声明时可以定义显式地一个全局名字。它必须是**以\开头以空格结尾**

```
export "DPI-C" f_plus = function \f+ ; // "f+" exported as "f_plus"
export "DPI-C" function f; // "f" exported under its own name
import "DPI-C" init_1 = function void \init[1] (); // "init_1" is a linkage name
import "DPI-C" \begin = function void \init[2] (); // "begin" is a linkage name
```
相同C标识符的多个export 声明是允许，前提是它们在不同的scope里

## 35.5 imported tasks and functions 

### 35.5.1 属性 (properties) ##

** pure **
如果一个function的返回值只依赖于它的输入参数，且没有副作用， 那么它可以声明为pure
imported task绝不能声明为pure
** context **
一个imported subroutine要调用exported subroutines, 或要获取SV data objects（e.g., via VIP calls)而不是它的实际值， 那么需要声明它为context。
如果没有描述，那么subroutine应该不访问SV data objects; 不过它可以进行一些有副作用的行为，比如写文件，操作一个全局变量
compiler不会对这些properties进行检查, 所以使用它们要小心

#### 35.5.1.1 instant completion of imported functions
它们立即被执行，并且无耗时

#### 35.5.1.2 input, output, inout ###
*input*: 不会被修改
*ouput*: output的初始值是不确定的，所以imported function不应该依赖于它
*inout*: imported function可以得到inout变量的初始值。imported function对于inout参数的修改可以被function外面看到

#### 35.5.1.4 内存管理 ###
外部语言和SV各自负责自己内存的申请和释放
一个比较复杂且允许的情况是一个imported function申请了一块内存并把handle传到SV里， 然后SV调用另一个imported function来释放它

### 35.5.1.5 重入 (Reentrancy of imported tasks)
对于imported task的调用可能会导致自身进程暂停。这种情况发生在imported task又调用带有delay或event wait的exported task时。
所以有可能一个imported C code把多个线程同时激活。关于标准的重入规则是由C端决定。可以使用一些标准的多线程安全库，或者静态变量来做控制

#### 35.5.1.6 C++异常 (C++ exceptions)
可以使用C++，但前提是在language边界遵守C链接约定。
C++异常不应该传播到subroutine以外， 如果异常传递到SV里，那么这是一个未定义的行为

### 35.5.2 Pure functions
pure function特点之一是如果它的返回不需要那么可以把pure function调用去掉。或者它的输入没变，那么可以不用重新计算。
只有没有output和inout形参的nonvoid function可以被声明为pure
声明为pure的function没有任何副作用
返回只依赖于输入值
对于这种funciton的调用可以被SV compiler优化。或者当input没变时，可以直接使用之前的值
pure function应该不直接或间接（比如调用其他function)执行下面动作：
* 文件操作
* 对任何事物的读写，包括I/O, 环境变量，来自操作系统，程序，进程的对象， shared memory, socket等
* 访问任何永久变量，比如全局的或静态的变量

### 35.5.3 context tasks and functions
当imported subroutine要求调用上下文需要被知道时，调用者要采取特别的指令来提供这样的上下文。特殊的指令比如创建一个指向目前实例的内部变量。
当有当context被显式指定时，才会做这样的instrumentation。

exported subroutine必须知道它被调用时的上下文， 包括当它们被imported subroutine调用时。
imported subroutine可以在调用exported subroutine前，先调用svSetScope来显式地设定上文。否则，exported sobroutine的上下文就是调用import subroutine时的实例所在上下文。由于一个imported subroutine会存在于多个实例化后的作用域(scope)里， 所以展开(elaboration)后会有多个exported subroutine的实例(instances).如果不调用svSetScope, 那么这些exported instances的上下文就是调用imported subroutine的作用域(instantiated scope) 

外部语言可以通过一些其他接口(如 VPI callback)来调用svSetScope或其他DPI相关的scope APIs， 然后也可以在一个指定的作用域(instantiated scope）里调用exported subroutine。
DPI的scope相关APIs的行为和DPI exported subroutine的调用由各simulator决定，DPI spec不做规定

在SV里最开始调用imported subroutine的地方称为调用链初始点（root of the call chain)

上下文属性会从SV里应用到每一个imported subroutine. 这意味着根部的或调用链中间的imported call不一定能够把它的上下文环境传递到它的下一个import call.
所以一个无上下文环境的imported subroutine不能够调用一个SV exported subroutine. 这样的行为会导致出错

下面是关于imported call chain的一些特性：
- 下面动作决定导入调用链(import call chain)的上下文值：
    - 当SV subroutine调用一个导入DPI subroutine， 一个导入声明所在的实例化作用域的上下文就为这个导入调用链(imprt call chain)创建好了
    - 当处于导入调用链(import call chain)中的一个routine调用svSetScope并传入一个合法参数时，调用链的上下文就被设置成svSetScope参数所示上下文
    - 当一个导入调用链中调用一个导出的SV subroutine完成并返回时，调用链的上下文回到调用前的值
- 仿真器需要管理DPI的上下文，需要检测控制权如何在SV和外部语言间传递。如果用户代码里通过如C里的setjmp/longjmp等结构来从一个导出的调用链(SV)回到它的导入调用链的调用者(C), 那么它的结果是未定义的。（例如：CA -> SVA -> CB, CB里通过setjmp/longjmp回到CA）
- 一个特定的import subroutine是否上下文相关是由它自身声明时的context属性决定的。这个属性不会传递到它的后续的import function （译者注：这个地方是function而不是subroutine?）
- import call的上下文特性无法动态改变
- 上下文特性和调用链绑定，而不是和imported subroutine; 因此，一个subroutine可以在一个调用链中是context，而在另一个调用链中non-context。

一个没有定义为context的imported subroutine只访问它的实现参数。所以不是仿真器优化的障碍。而context的imported subroutine能够通过VPI和export subroutine来访问任何SV data objects， 所以会造成SV compiler的优化障碍。

只有context imported subroutine是被特殊装置地，并只做保守的优化。只有这种subroutine可以在安全地其他subroutines, 包括VPI或exported SV subroutines; 否则调用VPI和exported SV subroutine的行为是不可预测的，如果被调用者需要的上下文没有普查正确设置，那么会造成崩溃。定义一个context import subroutine并不会使simulator的其他接口自动可用。比如VPI访问还是依赖于正确的实现机制。DPI 调用不会自动创建或提供任何句柄或特定环境给其他接口用。这个是用户的责任。

context imported subroutines总是隐性地有一个完整实例名的作用域（scope）。这个作用域定义了哪些SV subroutines可以被importored subroutine直接调用；也只有这此在相同作用域的exported subroutine可以被直接调用。如果要调用其他的exported SV subroutine, imported subroutine需要先修改它的目前作用域。

相关的DPI functions可以允许imported subroutines来获取或接口用它的作用域, 如svGetScope(), svSetScope, svGetNameFromScope, svGetScopeFromName。

### 35.5.4 Import declarations

所有imported subroutine都需要被声明。
imprted subroutine和SV的subroutine相似，可以有0个或多个input, output或inout形参。imported functions可以返回一个或0个(void)值。imported tasks总是返回一个int值，在外部语言里相当于一个int function。
（译者： 原文中说是作为DPI disable protocol的一部分，没搞明白)

```verilog
dpi_import_export ::=                                                         // from A.2.6
        import dpi_spec_string [ dpi_function_import_property ] [ c_identifier = ] dpi_function_proto ;
        | import dpi_spec_string [ dpi_task_import_property ] [ c_identifier = ] dpi_task_proto ;
        | export dpi_spec_string [ c_identifier = ] function function_identifier ;
        | export dpi_spec_string [ c_identifier = ] task task_identifier ;
dpi_spec_string ::= "DPI-C" | "DPI"
dpi_function_import_property ::= context | pure
dpi_task_import_property ::= context
dpi_function_proto  ::= function_prototype
dpi_task_proto ::= task_prototype
function_prototype ::= function data_type_or_void function_identifier [ ( [ tf_port_list ] ) ]
task_prototype ::= task task_identifier [ ( [ tf_port_list ] ) ]             // from A.2.7
```

21) dpi_function_proto return types are restricted to small values, per 35.5.5
22)  Formals of dpi_function_proto and dpi_task_proto cannot use pass by reference mode and class types cannot be
passed at all; see 35.5.6 for a description of allowed types for DPI formal arguments.

import 声明描述了subroutine名， function返回值类型，和它的形参的类型和方向。它也能够提供形参的默认值。形参名是可选的，除非需要按名称绑定参数。
imported function可以有**context**或**pure**属性； imported tasks可以有**context**属性

导入声明相当于定义一个subroutine名， 所以多个相同的subroutine名是不允许导入同一作用域的。

- dpi_spec_string*可以取"DPI-C"或“DPI”。“DPI"是用于指示使用不建议用的sv packed array传输语义。这种语义下，参数是使用它在仿真器里的表示法来传递，而不是规一化的形式。

- c_identifier*是这个subroutine在外部语言里的链接名(linkage name). 如果没有提供的话就和它的SV subroutine名相同。
对于一个c_identifier, 所有声明都必须使用相同的类型签名(type signature), 包括返回值类型， 每个参数的数目， 顺序， 方向， 类型。类型包括维数，数组的边界和维数。类型签名也包括**pure/context**限定符， 和dpi_spec_string的值。

在多个不同作用域中可以有同样imported或exported subroutine的多个不同声明；因此， 参数名和默认值可能不同。

正式的参数名要区别packed和unpacked数组维数。

限定符**ref**不能用在import声明中。实际的实现方法只决取于外部语言层。实现方法对于SV边是透明的。

下面是一些声明的例子：
``` verilog
import "DPI -C " function void myInit();

// from standard math library
import "DPI -C " pure function real sin(real);

// from standard C library: memory management
import "DPI -C " function chandle malloc(int size); // standard C function
import "DPI -C " function void free(chandle ptr); // standard C function

// abstract data structure: queue
import "DPI -C " function chandle newQueue(input string name_of_queue);

// Note the following import uses the same foreign function for
// implementation as the prior import, but has different SystemVerilog name
// and provides a default value for the argument.
import "DPI -C " newQueue=function chandle newAnonQueue(input string s=null);
import "DPI -C " function chandle newElem(bit [15:0]);
import "DPI -C " function void enqueue(chandle queue, chandle elem);
import "DPI -C " function chandle dequeue(chandle queue);

// miscellanea
import "DPI -C " function bit [15:0] getStimulus();
import "DPI -C ” context function void processTransaction(chandle elem, output logic [64:1] arr [0:63]);
import "DPI -C " task checkResults(input string s, bit [511:0] packet)
```
### 35.5.5 Function result
imported function的返回值类型被限定为"small values":
* void , byte , shortint , int , longint , real , shortreal , chandle , and string
* Scalar values of type bit and logic
同样的限定也适用于exported function的返回值类型上

### 35.5.6 Types of formal argument (形参类型)

SV中丰富的数据类型可以被作为形参用于imported和export的例程。通常，C兼容的类型，packed类型，用户定义的类型和他们的组合都可以用作DPI例程的形参。
下面是SV中允许的所有可以作为DPI例程形参的类型：

- void, byte, shortint, int, longint, real, shortreal, chandle, time, integer, string
- bit, logic
- 由bit, logic组成的packed数组，结构体或联合体。在外部语言侧，所有packed类型数据都被译成1维的数组。
- 枚举类型被解释成同枚举本身相同的类型
- 新类型可以同下面几种方法构建
    - struct
    - union (packed only)
    - Unpacked array
    - typedef

下面是一些注意事项：
- 枚举类型不能被直接支持。代替地，枚举数据的实际类型被使用
- SV没有规定 packed或unpacked结构体，数组的实际内存存放结构。unpacked数据和它们的打包方式有管，而打包方式和C编译器有关。
- 在导出的DPI例程中， 不允许使用动态数组做为形参
- SV数据类型在内存中的实际存放结构对于SV语义来讲是透明的。它和外部语言有关。但我们可以大约知道SV数据类型是如何实现的。除了unpacked array，对导入例程的形参类型并没有什么限制。SV实现方式限制了哪些unpacked arrays会被当成固定大小的数组。（虽然它也可以被传输成非固定大小的open array). 实际的变量和实现相关，而open array提供了一种实现无关的方法。

#### 35.5.6.1 Open arrays

无论是packed维，或是unpacked维，或者两者都可以保留为空。这样的数据就叫open array. Open array提供了一种针对不同数据大小的通用解决方案。
导入函数的形参可以被描述成open array. 而导出函数不能够使用它作为形参。open array是唯一的一种放松参数匹配规则的方法。实际参数就可以处理不同大小的数据，从而达到代码通用化的目标。
虽然数组的packed部分可以是任意大小的，形参的变长维只能有一个是packed维。 这不是非常严厉的， packed类型都等价于一个一维packed数组，所以只有一个是可变的就相当于整个都可变。而unpacked维则没有限制。
如果形参有一个变长的packed维， 那么它将匹配任何packed维的实参。如果形参的unpacked维是变长的，那么它要求实参拥有一样的维数，只不过对应的具体维上长度是可变的。
下面是一些合法形参的例子：

```verilog
logic
bit [8:1]
bit []
bit [7:0] array8x10 [1:10]
logic [31:0] array32xN []
logic [] arrayNx3 [3:1]
bit [] arrayNxN []
```

下面是导入函数声明例子

```verilog
import "DPI-C" function void f1(input logic [127:0]);
import "DPI-C" function void f2(logic [127:0] i []); //open array of 128-bit
```

下面是利用open array来匹配不同长度实参的例子

```verilog
typedef struct {int i; ...} MyType;
import "DPI-C" function void f3(input MyType i [][]);

MyType a_10x5 [11:20][6:2];
MyType a_64x8 [64:1][-1:-8];

f3(a_10x5);
f3(a_64x8);
```

## 35.6 调用导入函数

调用导入函数和使用SV原生函数是一样的。所以导入函数声明和SV原生函数是一样的规范。特别地，带有默认值的参数在调用时可以省略；如果形参命名则可以用名字来做绑定。

### 35.6.1 变量传递

变量传递的规则是“所见即所得"。形参的计算顺序遵循SV通用规则。
变量的兼容性和强制转换都与SV原生函数一样。如果需要进行强制转换，那么会创建临时变量，并把它传递给实际变量。对于input和inout变量， 临时变量会被初始化成强制转换后的值。对于output和inout变量，实际值会被强制转换后然后赋给临时变量。临时变量和实际值之间的赋值遵循SV的赋值和强制转换通用规则。
在SV侧， 导入函数的输入值和调用者无关。形参输出值在开始是末定义的，然后被实际值赋值（可能发生强制转换）。导入函数不会改变输入变量的原始值。
在SV侧，变量传递的语义是： input变量是"copy-in", output变量是”copy-out", inout变量是"copy-in"和“copy-out"。 这里的"copy-in"和”copy-out"不是暗示它们的实际实现方式，而是指假想的赋值。
变量传递的真正实现对SV侧是透明的。SV并不清楚它实际上是值传递还是引用传递。它的实际实现方式是由外部语言定义的。

#### 35.6.1.1 WYSIWYG原则

WYSIWYS（所见即所得）原则保证了导入函数的形参类型：实际变量被要求要和形参完全相同，除了open array. 形参只和导入函数的声明现场有关。
没有编译器（C或者SV）可以在形参和实参类型间做强制转换。调用者的形参是被定义在另外一种语言里，所以它们相互间无法可见。用户需要理解并保证它们是匹配的类型。
open array中的可变长维度会有实参的实际长度。形参中的变长，packed的维度会拥有实际参数的范围。变长， packed维度会被归一化 （[15:8] -> [8:0]). 变长范围在调用现场才能确定。其他信息都在函数声明中定义了。

因此， 如果一个形参被定义成bit [15:8] b [], 那么它描述了一个unpacked array, 它的每个元素也是一个packed bit array, 数据绑定范围是15到8. 实际参数的长度会被绑定到它的unpacked部分。
有时候允许传递一个动态数组会为导入function和task的实参。原则同SV里传递动态数组给原生function和task。

### 35.6.2 output和inout变量的值变化

仿真器负责处理output和inout变量的值变化。这样的值改变会被仿真器探测到，并被返回到SV代码。
对于output和inout变量， 值传播会立即发现一旦控制权从导入函数返回。如果有多个变量返回，那么值传播的顺序遵循SV的通用规则。

## 35.7 导出函数

DPI允许其他语言调用SV函数。函数和变量兼容性要求和导入函数一样。声明SV函数被导出并不改变它的语义和行为；没有什么副作用除了当它被用于DPI 调用链中（比如调用者又被SV导入并调用）。

可以被其他语言调用的SV函数需要被声明为**export**。 声明必须在函数被定义的作用域里。每个函数只能被export声明一次。
一个重要的限制是： 类的成员函数不能被导出， 其他都可以。
同导入声明一样， 卖出声明可以定义一个可选的C标识符（用于其他语言）

```
dpi_import_export ::=         //from A.2.6
     ...
     | export dpi_sepc_string [c_identifier=] function function_identifier;
     ...
dpi_spec_string ::= "DPI-C" | "DPI"
```

## 35.8 导出任务

SV允许外部语言可以调用tasks. 同函数一样， 这样的task被定义为导出taks。
上面对于导出函数的所有描述都可应用于导出task。包括合法的定义作用域和可选的C标识符。

从一个导入的函数里调用一个导出的task是非法的。这个语义和原生SV语义是一致的，函数不能调用task。

只有导入的task拥有**context**属性时才可以被导入的task调用。

导出task和导出函数的一个不同处是SV task没有返回值。导出task的返回值是一个int类型值，用来指示是否有disable用于当前线程。
相似的，导入task也会返回一个int类型值来的说明导入task是否收到一个disable.

## 35.9 禁用DPI task和function

使用disable语句可以禁用正在执行的DPI调用链。当导入的例程被禁用时，C代码会遵循一个简单的关闭协议。协议会允许C代码去执行必要的资源释放清理工程，比如关闭文件句柄，关闭VPI句柄，释放堆内存。

当一个disable语句作用于某个目标或它的父范围域，那么我们就称这个导入task或function在disable状态。一个导入task或function如果调用了一个导出的task或function， 那么在进入disable状态前我必须从调用中返回。这个协议的关键之处在于被关闭的导入task或function将确认他们被关闭了。task或function可以通过svIsDisabledState()来确定它是否在disable状态。

这个协议由如下部分组成：
a) 当一个导出的task被disable时，返回1， 否则返回0；
b) 当一个导入的task被disable时，返回1， 否则返回0
c) 当导入的function被disable时， 它将会先调用svAckDisabledState()
d) 一旦导入的task或function进入disable状态，它将不允许再调用任何导出的task或function

b), c), d)是强制行为。DPI的输写者需要确保正常的行为。
a)是由SV仿真器来保证的。此外仿真器也会检查b), c), d)是否被遵守。如果任何协议没被遵守，仿真器应该提示致命错误。

DPI的另外一侧包含一个禁用协议，该协议由用户代码与仿真器一起实现。禁用协议允许外部模型参与SystemVerilog禁用处理。参与方法是通过DPI task的特殊返回值和特殊API调用来完成。
特殊的返回值不需要更改SV代码中导入或导出DPI task的调用语法。虽然仿真器保证了导出task的返回值，但对于导入task，DPI另一侧必须确保返回正确的值。
对导入task的调用与对SV原生task的调用是无法区分的。同样，对代码中的导出task的调用与对非SV task的调用是无法区分的。
如果导出的task本身是禁用的目标，则当导出task返回时，其父项导入的task不被视为处于disable状态。在这种情况下，导出的任务应返回值0，并且对svIsDisabledState()的调用也应返回0。
当DPI导入的子例程由于被禁用而返回时，其输出和inout参数的值未定义。同样，当导入的函数由于禁用而返回时，函数返回值是不确定的。 C程序员可以从禁用的函数中返回值，而C程序员可以将任意值写入导入例程的output和inout参数位置。但是，如果禁用有效，SV仿真器没有义务将任何此值传播到调用SystemVerilog代码中。


