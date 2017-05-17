# 需求
随着时代的发展，现在的芯片规模越来越大，哪怕模块级的验证环境也需要相当长的build时间，各种仿真工具也在改进编译和运行性能，还发明了增量编译。但无论如何turnaround的时间还是比较长，而且方法越复杂越容易出错。而DPI-C则比较简单，能够解决某些场景下的问题。

# 适用范围
DPI-C比较适用于SV和外部语言间的“简单数据“交互

# 简介
DPI全称Direct programming interface. SV和外部语言对彼此是透明的。理论上DPI可以支持很多语言，但目前SV只支持C
DPI遵循black-box原则。描述和实现隔离。带来的好处就是在进行sv编译时我们只需要定义好*描述*, 而实现可以”延迟决定“。
我把使用DPI的进行灵活环境搭建的方法学称为面向接口编程。
SV和C通过function这个封装单元来交互。这个function可以定义在SV或C里。

# tasks/functions
functions在外部实现，被SV调用，这种functions叫**imported functions**.
functions在SV实现，被外部调用， 这种functions叫**expored functions**
DPI允许通过function的参数（arguments）和返回值(result)来在两个域间传递**SV data**

外部代码也可调用SV tasks, 原生的SV代码也可能调用imported tasks.
imported tasks和原生的SV task在语义上是一样的：
* 无返回值
* 可以消耗仿真时间

所有DPI functions都是不消耗仿真时间的，DPI无法提供除数据交换和控制权转移以外的同步手段.

每个imported的subroutine都必须先声明。
import声明可以写在任何SV subrountine允许的地方。同一个外部subrountine可以被多处调用。
imported subroutine可以有0个可多个 input, output, inout 参数。
imported tasks只能返回void value; imported function可以返回值或void value.

# Data types
SV Data types必须是能够跨越SV和外部语言的类型, 称为富类型：
* void, byte, shortint, int, longint, real, shortreal, chandle, string
* scalar value of bit and logic
* packed arrays, structs, uniions composed of types bit and logic
* emumeration types (interpreted as the associated real type)
* types constructed from the following constructs:
    * struct
    * union(packed only)
    * unpacked array
    * typedef

下面是一些注意点：
* enum 不能被直接支持，实际上会被解释为enum type关联的类型
* 在exported DPI subroutine里， 声明形参为dynamic array容易出错
* SV data的实际内存结构对于SV是透明的。
Function的返回值必须是small values:
* void, byte, shortint, int, longint, real, shortreal, chandle, string
* scalar value of bit and logic
而imported function的形参可以是open arrays

# 内存数据结构
DPI并没有对于数据在内存里的存在形式做任何约束， 所以这是平台相关的，和SV无关

# DPI的两层
##SV layer
不依赖于外面到底是什么语言
##foreign language layer
定义实参是如何被传递的， SV data（如logic, packed）是如何表示的。如何与类C的类型转换
对于不同的外部语言，SV端的代码应该是一样的

#imported and exported functions的全局命名空间
每个imported subrountine最终最会有一个全局符号标志；而每个exported subroutine会定义一个全局符号标志。
它们必须唯一。它们遵循C的命名规则，必须以字母或_开头。
import和export声明时可以定义显式地一个全局名字。它必须是**以\开头以空格结尾**

```
export "DPI-C" f_plus = function \f+ ; // "f+" exported as "f_plus"
export "DPI-C" function f; // "f" exported under its own name
import "DPI-C" init_1 = function void \init[1] (); // "init_1" is a linkage name
import "DPI-C" \begin = function void \init[2] (); // "begin" is a linkage name
```
相同c_identifier的多个export 声明是允许，前提是它们在不同的scope里

# imported tasks and functions #
## 属性 (properties) ##
* pure
如果一个function的返回值只依赖于它的输入参数，且没有副作用， 那么它可以声明为pure
imported task绝不能声明为pure
* context
一个imported subroutine要调用exported subrountines, 或要获取SV data objects（e.g., via VIP calls)而不是它的实际值， 那么需要声明它为context。
如果没有描述，那么subroutine应该不访问SV data objects; 不过它可以进行一些有副作用的行为，比如写文件，操作一个全局变量
compiler不会对这些properties进行检查, 所以使用它们要小心
## instant completion of imported functions
它们立即被执行，并且无耗时
## input, output, inout ##
* input: 不会被修改
* ouput: output的初始值是不确定的，所以imported function不应该依赖于它
* inout: imported function可以得到inout变量的初始值。imported function对于inout参数的修改可以被function外面看到
## 内存管理 ##
外部语言和SV各自负责自己内存的申请和释放
一个比较复杂且允许的情况是一个imported function申请了一块内存并把handle传到SV里， 然后SV调用另一个imported function来释放它
## 重入 (Reentrancy of imported tasks)
对于imported task的调用可能会导致自身进程暂停。这种情况发生在imported task又调用带有delay或event wait的exported task时。
所以有可能一个imported C code把多个线程同时激活。关于标准的重入规则是由C端决定。可以使用一些标准的多线程安全库，或者静态变量来做控制
## C++异常 (C++ exceptions)
可以使用C++，但前提是在language边界遵守C链接约定。
C++异常不应该传播到subroutine以外， 如果异常传递到SV里，那么这是一个未定义的行为
## Pure functions
pure function特点之一是如果它的返回不需要那么可以把pure function调用去掉。或者它的输入没变，那么可以不用重新计算。
只有没有output和inout形参的nonvoid function可以被声明为pure
声明为pure的function没有任何副作用
返回只依赖于输入值
对于这种funciton的调用可以被SV compiler优化。或者当input没变时，可以直接使用之前的值
pure function应该不直接或间接（比如调用其他function)执行下面动作：
* 文件操作
* 对任何事物的读写，包括I/O, 环境变量，来自操作系统，程序，进程的对象， shared memory, socket等
* 访问任何永久变量，比如全局的或静态的变量
## context tasks and functions
当imported subrountine要求调用上下文需要被知道时，调用者要采取特别的指令来提供这样的上下文。特殊的指令比如创建一个指向目前实例的内部变量。
当有当context被显式指定时，才会做这样的instrumentation。

exported subrountine必须知道它被调用时的上下文， 包括当它们被imported subroutine调用时。
imported subrountine可以在调用exported subrountine前，先调用svSetScope来显式地设定上文。否则，exported sobrountine的上下文就是调用import subrountine时的实例所在上下文。由于一个imported subrountine会存在于多个实例化后的作用域(scope)里， 所以展开(elaboration)后会有多个exported subroutine的实例(instances).如果不调用svSetScope, 那么这些exported instances的上下文就是调用imported subroutine的作用域(instantiated scope) 

外部语言可以通过一些其他接口(如 VPI callback)来调用svSetScope或其他DPI相关的scope APIs， 然后也可以在一个指定的作用域(instantiated scope）里调用exported subrountine。
DPI的scope相关APIs的行为和DPI exported subrountine的调用由各simulator决定，DPI spec不做规定

在SV里最开始调用imported subrountine的地方称为调用链初始点（root of the call chain)

上下文属性会从SV里应用到每一个imported subrountine. 这意味着根部的或调用链中间的imported call不一定能够把它的上下文环境传递到它的下一个import call.
所以一个无上下文环境的imported subrountine不能够调用一个SV exported subrountine. 这样的行为会导致出错

下面是关于imported call chain的一些特性：
* 下面动作决定导入调用链(import call chain)的上下文值：
    * 当SV subrountine调用一个导入DPI subrountine， 一个导入声明所在的实例化作用域的上下文就为这个导入调用链(imprt call chain)创建好了
    * 当处于导入调用链(import call chain)中的一个rountine
   
