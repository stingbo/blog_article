__注:__
转载文章翻译自Nikita的文章，欢迎指正[查看原文](http://nikic.github.io/2017/04/14/PHP-7-Virtual-machine.html)

本文旨在提供PHP7中Zend虚拟机(Zend Virtual Machine)的一个概述。这不是一个全面的描述，但我尽量涵盖大部分重要内容，以及一些细腻的细节。

此描述针对PHP版本7.2(目前正在开发中)，但几乎所有内容也适用于PHP7.0/7.1。然而，与PHP5.x系列VM的也有很大不同，我一般不会花大力气去描述相似之处。

这篇文章大部分是在讨论指令列表级别的内容，只有最后几个部分涉及到VM的实际C级实现。但是，我非常希望提供一些构成VM的主要文件的链接：

* [zend_vm_def.h](https://github.com/php/php-src/blob/master/Zend/zend_vm_def.h)：VM的定义文件。
* [zend_vm_execute.h](https://github.com/php/php-src/blob/master/Zend/zend_vm_execute.h)：已生成的虚拟机。
* [zend_vm_gen.php](https://github.com/php/php-src/blob/master/Zend/zend_vm_gen.php)：VM生成脚本。
* [zend_execute.c](https://github.com/php/php-src/blob/master/Zend/zend_execute.c)：大多数可直接运行代码。

##### 操作码(Opcodes)

最开始是opcode(操作码)。“Opcode”是指我们如何引用一个完整的VM指令(包括操作数)，也可以仅指“实际”的操作代码(operation code)，“Opcode”是一个决定指令类型的小整数。这应该可以从上下文中弄清楚。在源码中，完整的指令通常被称为“oplines”。

单个指令符合以下`zend_op`结构：
```
struct _zend_op {
    const void *handler;
    znode_op op1;
    znode_op op2;
    znode_op result;
    uint32_t extended_value;
    uint32_t lineno;
    zend_uchar opcode;
    zend_uchar op1_type;
    zend_uchar op2_type;
    zend_uchar result_type;
}
```

因此，操作码本质上是“三地址码”指令格式。有一个操作码(`opcode`)确定指令类型，有两个输入操作数`op1`和`op2`以及一个输出操作数`result`。

不是所有指令都使用到这三个操作数。`ADD`指令(代表`+`运算符)使用到这全部三个。`BOOL_NOT`指令(表示`!`运算符)仅使用`op1`和`result`。`ECHO`指令仅使用`op1`。某些指令也可能使用或不使用操作数。例如，`DO_FCALL`可能有也可能没有`result`操作数，具体取决于函数是否有返回值。某些指令需要两个以上的输入操作数，在这种情况下，将使用第二个伪指令(`OP_DATA`)来传递额外的操作数。

除了这三个标准操作数，还有一个额外的数值`extended_value`字段，可用于保存其他指令修饰符。例如，对于`CAST`它可能包含要转换的目标类型。

每个操作数都有一个类型，分别存储在`op1_type`，`op2_type`和`result_type`中。可能的类型是`IS_UNUSED`，`IS_CONST`，`IS_TMPVAR`，`IS_VAR`和`IS_CV`。

后三种类型指定变量操作数(具有三种不同类型的VM变量)，`IS_CONST`表示常量操作数(`5`或`“字符串”`或甚至`[1,2,3]`)，而`IS_UNUSED`表示实际未使用的操作数，或者用作32位数字型值(“立即”，用汇编术语表示)。例如，跳转指令将跳转目标存储在`UNUSED`操作数中。

##### 获取操作码转储(Obtaining opcode dumps)

-----------#TODO
在下文中，我会经常写出PHP为某些示例代码生成的操作码序列。目前有三种方法可以获得这样的操作码转储：
```
# Opcache, since PHP 7.1
php -d opcache.opt_debug_level=0x10000 test.php

# phpdbg, since PHP 5.6
phpdbg -p* test.php

# vld, third-party extension
php -d vld.active=1 test.php
```

其中，opcache提供最高质量的输出。 本文中使用的列表基于opcache转储，并进行了少量语法调整。 幻数0x10000是“优化前”的缩写，因此我们可以看到PHP编译器生成的操作码。 0x20000会为您提供优化的操作码。 Opcache还可以生成更多信息，例如0x40000将生成CFG，而0x200000将生成类型和范围推断的SSA形式。 但这已经超越了我们自己：简单的旧线性化操作码转储就足以满足我们的需求。

##### Variable types

在处理PHP虚拟机时，可能最重要的一点是它使用的三种不同的变量类型。在PHP 5中，TMPVAR，VAR和CV在VM堆栈上具有非常不同的表示形式，以及访问它们的不同方式。在PHP 7中，它们变得非常相似，因为它们共享相同的存储机制。但是，它们可以包含的值和它们的语义存在重要差异。

CV是“编译变量”的缩写，指的是“真正的”PHP变量。如果函数使用变量$ a，则$ a将有相应的CV。

CV可以具有UNDEF类型，以表示未定义的变量。如果在指令中使用UNDEF CV，它将(在大多数情况下)抛出众所周知的“未定义变量”通知。在函数入口上，所有非参数CV都被初始化为UNDEF。

CV不会被指令消耗，例如指令ADD $ a，$ b不会破坏存储在CV $ a和$ b中的值。相反，所有CV都在范围退出时被一起销毁。这也意味着所有CV在整个函数期间都是“实时”的，其中“实时”在这里指的是包含有效值(在数据流意义上不存在)。

另一方面，TMPVAR和VAR是虚拟机临时工。它们通常作为某些操作的结果操作数引入。例如，代码$ a = $ b + $ c + $ d将导致类似于以下的操作码序列：

```
T0 = ADD $b, $c
T1 = ADD T0, $d
ASSIGN $a, T1
```

TMP / VAR总是在使用前定义，因此不能保持UNDEF值。与CV不同，这些变量类型被它们使用的指令所消耗。在上面的例子中，第二个ADD将破坏T0操作数的值，并且在此点之后不得使用T0(除非事先写入) 。同样，ASSIGN将消耗T1的值，使T1无效。

因此，TMP / VAR通常非常短暂。在许多情况下，临时只存在于单个指令的空间中。在这个短暂的活跃时间间隔之外，临时值是垃圾。

那么TMP和VAR有什么区别？不多。这种区别继承自PHP 5，其中TMP是分配的VM堆栈，而VAR则是堆分配的。在PHP 7中，所有变量都是堆栈分配的。因此，现在TMP和VAR之间的主要区别在于只允许后者包含REFERENCE(这允许我们在TMP上省略DEREF)。此外，VAR可以包含两种类型的特殊值，即类条目和INDIRECT值。后者用于处理非平凡的任务。

下表试图总结主要差异：

|       | UNDEF | REF | INDIRECT | Consumed? | Named? |
|:------|:-----:|:---:|:--------:|:---------:|:------:|
|CV     |  yes  | yes |    no    |     no    |  yes   |
|TMPVAR |   no  |  no |    no    |    yes    |   no   |
|VAR    |   no  | yes |   yes    |    yes    |   no   |

##### Op arrays

所有PHP函数都表示为具有公共zend_function头的结构。 这里的“功能”应该被广泛地理解，并且包括从“真实”功能，方法，到独立的“伪主”代码和“评估”代码的所有内容。

Userland函数使用zend_op_array结构。 它有30多个成员，所以我现在开始使用简化版本：
```
struct _zend_op_array {
    /* Common zend_function header here */

    /* ... */
    uint32_t last;
    zend_op *opcodes;
    int last_var;
    uint32_t T;
    zend_string **vars;
    /* ... */
    int last_literal;
    zval *literals;
    /* ... */
};
```

这里最重要的部分当然是操作码，它是一个操作码(指令)数组。 last是此数组中的操作码数。 请注意，这里的术语令人困惑，因为最后听起来应该是最后一个操作码的索引，而它实际上是操作码的数量(比最后一个索引大一个)。 这同样适用于op阵列结构中的所有其他last_ *值。

last_var是CV的数量，T是TMP和VAR的数量(在大多数地方我们没有对它们进行强有力的区分)。 CV中名称数组的变量。

literals是代码中出现的文字值数组。 这个数组是CONST操作数引用的。 根据ABI，每个CONST操作数将存储指向此文字表的指针，或者存储相对于其开始的偏移量。

op数组结构比这更多，但它可以等待以后。

##### Stack frame layout

除了一些执行程序全局变量(EG)之外，所有执行状态都存储在虚拟机堆栈中。 VM堆栈以256 KiB的页面分配，并且各个页面通过链接列表连接。

在每个函数调用上，在VM堆栈上分配新的堆栈帧，具有以下布局：
```
+----------------------------------------+
| zend_execute_data                      |
+----------------------------------------+
| VAR[0]                =         ARG[1] | arguments
| ...                                    |
| VAR[num_args-1]       =         ARG[N] |
| VAR[num_args]         =   CV[num_args] | remaining CVs
| ...                                    |
| VAR[last_var-1]       = CV[last_var-1] |
| VAR[last_var]         =         TMP[0] | TMP/VARs
| ...                                    |
| VAR[last_var+T-1]     =         TMP[T] |
| ARG[N+1] (extra_args)                  | extra arguments
| ...                                    |
+----------------------------------------+
```

该帧以zend_execute_data结构开头，后跟一个可变插槽数组。 插槽都是相同的(简单的zval)，但用于不同的目的。 第一个last_var槽是CV，其中第一个num_args包含函数参数。 CV插槽后面跟着TMP / VAR的T插槽。 最后，有时可以在帧的末尾存储“额外”参数。 这些用于处理func_get_args()。

指令中的CV和TMP / VAR操作数被编码为相对于堆栈帧开始的偏移，因此获取某个变量只是从execute_data位置读取的。

帧开始处的执行数据定义如下：
```
struct _zend_execute_data {
    const zend_op       *opline;
    zend_execute_data   *call;
    zval                *return_value;
    zend_function       *func;
    zval                 This;             /* this + call_info + num_args    */
    zend_class_entry    *called_scope;
    zend_execute_data   *prev_execute_data;
    zend_array          *symbol_table;
    void               **run_time_cache;   /* cache op_array->run_time_cache */
    zval                *literals;         /* cache op_array->literals       */
};
```

最重要的是，这个结构包含opline，它是当前执行的指令，而func是当前执行的函数。 此外：

* return_value是指向将存储返回值的zval的指针。
* 这是$ this对象，但也编码了一些未使用的zval空间中的函数参数和一些调用元数据标志。
* called_scope是static ::在PHP代码中引用的范围。
* prev_execute_data指向上一个堆栈帧，此函数在完成运行后将返回执行。
* symbol_table是一个典型未使用的符号表，用于某些疯狂的人实际使用变量或类似功能的情况。
* run_time_cache缓存op数组运行时缓存，以便在访问此结构时避免一个指针间接(稍后讨论)。
* literals以相同的原因缓存op数组文字表。

##### Function calls

我在execute_data结构中跳过了一个字段，即call，因为它需要一些关于函数调用如何工作的进一步上下文。

所有调用都使用相同指令序列的变体。 全局范围内的var_dump($ a，$ b)将编译为：
```
INIT_FCALL (2 args) "var_dump"
SEND_VAR $a
SEND_VAR $b
V0 = DO_ICALL   # or just DO_ICALL if retval unused
```

根据呼叫的类型，有8种不同类型的INIT指令。 INIT_FCALL用于调用我们在编译时识别的自由函数。 类似地，根据参数的类型和函数，有十种不同的SEND操作码。 只有少量的四个DO_CALL操作码，其中ICALL用于调用内部函数。

虽然具体说明可能不同，但结构始终相同：INIT，SEND，DO。 调用序列必须要解决的主要问题是嵌套函数调用，它们编译如下：
```
# var_dump(foo($a), bar($b))
INIT_FCALL (2 args) "var_dump"
    INIT_FCALL (1 arg) "foo"
    SEND_VAR $a
    V0 = DO_UCALL
SEND_VAR V0
    INIT_FCALL (1 arg) "bar"
    SEND_VAR $b
    V1 = DO_UCALL
SEND_VAR V1
V2 = DO_ICALL
```

我缩进了操作码序列，以便可视化哪些指令对应于哪个调用。

INIT操作码在栈上推送一个调用帧，它包含足够的空间用于函数中的所有变量以及我们知道的参数数量(如果涉及参数解包，我们可能会得到更多的参数)。这个调用帧是用被调用的函数$ this和called_scope初始化的(在这种情况下，后者都是NULL，因为我们正在调用自由函数)。

指向新帧的指针存储在execute_data-> call中，其中execute_data是调用函数的帧。在下文中，我们将这种访问表示为EX(调用)。值得注意的是，新帧的prev_execute_data被设置为旧的EX(调用)值。例如，调用foo的INIT_FCALL将prev_execute_data设置为var_dump的堆栈帧(而不是周围函数的堆栈帧)。因此，在这种情况下，prev_execute_data形成“未完成”调用的链接列表，而通常它将提供回溯链。

然后，SEND操作码继续将参数推送到EX(调用)的变量槽中。此时，参数都是连续的，并且可能会从为参数指定的部分溢出到其他CV或TMP中。这将在稍后修复。

最后，DO_FCALL执行实际调用。什么是EX(调用)成为当前函数，prev_execute_data重新链接到调用函数。除此之外，呼叫过程取决于它是什么类型的功能。内部函数只需要调用处理函数，而userland函数需要完成堆栈帧的初始化。

此初始化涉及修复参数堆栈。 PHP允许向函数传递比预期更多的参数(并且func_get_args依赖于此)。但是，只有实际声明的参数具有相应的CV。除此之外的任何参数都将写入为其他CV和TMP保留的内存中。因此，这些参数将在TMP之后移动，最终将参数分割为两个非连续的块。

要明确说明，userland函数调用不涉及虚拟机级别的递归。它们只涉及从一个execute_data到另一个execute_data的切换，但VM继续以线性循环运行。仅当内部函数调用userland回调(例如，通过array_map)时，才会发生递归虚拟机调用。这就是为什么PHP中的无限递归通常会导致内存限制或OOM错误的原因，但是可以通过回调函数或魔术方法的递归来触发堆栈溢出。

##### Argument sending

PHP使用大量不同的参数发送操作码，其差异可能令人困惑，不用多亏一些不幸的命名。

SEND_VAL和SEND_VAR是最简单的变体，它们处理在编译时已知为by-value的按值参数的发送。 SEND_VAL用于CONST和TMP操作数，而SEND_VAR用于VAR和CV。

相反，SEND_REF用于编译期间已知为引用的参数。由于只能通过引用发送变量，因此该操作码仅接受VAR和CV。

SEND_VAL_EX和SEND_VAR_EX是SEND_VAL / SEND_VAR的变体，用于我们无法静态确定参数是按值还是按引用的情况。这些操作码将根据arginfo检查参数的类型并相应地表现。在大多数情况下，不使用实际的arginfo结构，而是直接在函数结构中使用紧凑的位向量表示。

然后是SEND_VAR_NO_REF_EX。不要试图在它的名字中读取任何东西，它是完全撒谎。传递不是真正“变量”但确实将VAR返回到静态未知参数的东西时使用此操作码。使用它的两个特定示例是将函数调用的结果作为参数传递，或传递赋值的结果。

这个案例需要一个单独的操作码有两个原因：首先，如果你尝试通过ref传递类似的赋值，它将生成熟悉的“只有变量应该通过引用传递”的通知(如果使用SEND_VAR_EX代替它，那么它将是静默的允许)。其次，此操作码处理您可能希望将引用返回函数的结果传递给by-reference参数(不应抛出任何内容)的情况。此操作码的SEND_VAR_NO_REF变体(不带_EX)是我们静态知道引用是预期的情况的特殊变体(但我们不知道参数是否为1)。

SEND_UNPACK和SEND_ARRAY操作码分别处理参数解包和内联call_user_func_array调用。它们都将数组中的元素推送到参数堆栈，并且各种细节不同(例如，解包支持Traversable，而call_user_func_array则不支持)。如果使用解包/ cufa，则可能需要将堆栈帧扩展到其先前的大小(因为在初始化时不知道实际的函数参数数)。在大多数情况下，只需移动堆栈顶部指针即可实现此扩展。但是，如果这会跨越堆栈页面边界，则必须分配新页面，并且需要将整个调用框架(包括已经推送的参数)复制到新页面(我们无法处理跨页面的调用框架)边界)。

最后一个操作码是SEND_USER，用于内联call_user_func调用并处理其中的一些特性。

虽然我们还没有讨论过不同的变量获取模式，但这似乎是引入FUNC_ARG获取模式的好地方。考虑一个简单的调用，如func($ a [0] [1] [2])，我们在编译时不知道参数是通过-value还是by-reference传递的。在这两种情况下，行为都会大不相同。如果传递是按值并且$ a以前是空的，则可能必须生成一堆“未定义的索引”通知。如果传递是引用的，我们必须以静默方式初始化嵌套数组。

FUNC_ARG获取模式将通过检查当前EX(调用)函数的arginfo动态地选择两种行为之一(R或W)。对于func($ a [0] [1] [2])示例，操作码序列可能如下所示：
```
INIT_FCALL_BY_NAME "func"
V0 = FETCH_DIM_FUNC_ARG (arg 1) $a, 0
V1 = FETCH_DIM_FUNC_ARG (arg 1) V0, 1
V2 = FETCH_DIM_FUNC_ARG (arg 1) V1, 2
SEND_VAR_EX V2
DO_FCALL
```

##### Fetch modes

PHP虚拟机有四类fetch操作码：
```
FETCH_*             // $_GET, $$var
FETCH_DIM_*         // $arr[0]
FETCH_OBJ_*         // $obj->prop
FETCH_STATIC_PROP_* // A::$prop
```

这些正是人们期望他们做的事情，但需要注意的是基本的FETCH_ *变体仅用于访问变量和超全局变量：正常的变量访问通过更快的CV机制来代替。

这些获取操作码分为六种变体：
```
_R
_RW
_W
_IS
_UNSET
_FUNC_ARG
```

我们已经知道_FUNC_ARG在_R和_W之间选择，具体取决于函数参数是按值还是按引用。 让我们尝试创建一些我们希望出现不同fetch类型的情况：
```
// $arr[0];
V2 = FETCH_DIM_R $arr int(0)
FREE V2

// $arr[0] = $val;
ASSIGN_DIM $arr int(0)
OP_DATA $val

// $arr[0] += 1;
ASSIGN_ADD (dim) $arr int(0)
OP_DATA int(1)

// isset($arr[0]);
T5 = ISSET_ISEMPTY_DIM_OBJ (isset) $arr int(0)
FREE T5

// unset($arr[0]);
UNSET_DIM $arr int(0)
```

不幸的是，这产生的唯一实际获取是FETCH_DIM_R：其他所有内容都是通过特殊操作码处理的。 请注意，ASSIGN_DIM和ASSIGN_ADD都使用额外的OP_DATA，因为它们需要两个以上的输入操作数。 使用像ASSIGN_DIM这样的特殊操作码而不是类似于FETCH_DIM_W + ASSIGN的原因是(除了性能之外)这些操作可能被重载，例如，在ASSIGN_DIM情况下通过实现ArrayAccess :: offsetSet()的对象。 要实际生成不同的提取类型，我们需要提高嵌套级别：
```
// $arr[0][1];
V2 = FETCH_DIM_R $arr int(0)
V3 = FETCH_DIM_R V2 int(1)
FREE V3

// $arr[0][1] = $val;
V4 = FETCH_DIM_W $arr int(0)
ASSIGN_DIM V4 int(1)
OP_DATA $val

// $arr[0][1] += 1;
V6 = FETCH_DIM_RW $arr int(0)
ASSIGN_ADD (dim) V6 int(1)
OP_DATA int(1)

// isset($arr[0][1]);
V8 = FETCH_DIM_IS $arr int(0)
T9 = ISSET_ISEMPTY_DIM_OBJ (isset) V8 int(1)
FREE T9

// unset($arr[0][1]);
V10 = FETCH_DIM_UNSET $arr int(0)
UNSET_DIM V10 int(1)
```

在这里，我们看到尽管最外层访问使用专门的操作码，但嵌套索引将使用具有适当提取模式的FETCH来处理。 获取模式本质上有所不同：a)如果索引不存在，它们是否生成“未定义的偏移”通知，以及它们是否获取写入的值：

|      | Notice? | Write?   |
|:-----|:-------:|:--------:|
|R     |  yes    |  no      |
|W     |  no     |  yes     |
|RW    |  yes    |  yes     |
|IS    |  no     |  no      |
|UNSET |  no     |  yes-ish |

UNSET的情况有点奇怪，因为它只会获取现有的写入偏移量，并且单独留下未定义的偏移量。 正常的写入读取将初始化未定义的偏移量。

##### Writes and memory safety

Write fetches返回可能包含普通zval或另一个zval的INDIRECT指针的VAR。 当然，在前一种情况下，应用于zval的任何更改都不可见，因为该值只能通过VM临时访问。 虽然PHP禁止表达式如[] [0] = 42，但我们仍需要对call()[0] = 42这样的情况进行处理。取决于call()是按值返回还是按引用返回，这个表达式可能 或者可能没有可观察到的影响。

更典型的情况是fetch返回INDIRECT，其中包含指向正在修改的存储位置的指针，例如散列表数据数组中的某个位置。 不幸的是，这样的指针是脆弱的东西，很容易失效：对数组的任何并发写入都可能触发重新分配，留下一个悬空指针。 因此，防止在创建INDIRECT值的点与消耗它的位置之间执行用户代码至关重要。

思考以下例子：
```
$arr[a()][b()] = c();
```

将会产生：
```
INIT_FCALL_BY_NAME (0 args) "a"
V1 = DO_FCALL_BY_NAME
INIT_FCALL_BY_NAME (0 args) "b"
V3 = DO_FCALL_BY_NAME
INIT_FCALL_BY_NAME (0 args) "c"
V5 = DO_FCALL_BY_NAME
V2 = FETCH_DIM_W $arr V1
ASSIGN_DIM V2 V3
OP_DATA V5
```

值得注意的是，该序列首先从左到右执行所有副作用，然后才执行任何必要的写入提取(我们在此将FETCH_DIM_W称为“延迟的opline”)。 这确保了写入和消耗指令直接相邻。

思考另一个例子：
```
$arr[0] =& $arr[1];
```

这里我们有一些问题：必须获取赋值的两端以进行写入。 但是，如果我们为写入获取$ arr [0]然后为写入获取$ arr [1]，则后者可能使前者无效。 这个问题解决如下：

```
V2 = FETCH_DIM_W $arr 1
V3 = MAKE_REF V2
V1 = FETCH_DIM_W $arr 0
ASSIGN_REF V1 V3
```

这里首先获取$ arr [1]以进行写入，然后使用MAKE_REF将其转换为引用。 MAKE_REF的结果不再是间接的，不会失效，因此可以安全地执行$ arr [0]的获取。

##### 异常处理(Exception handling)

异常是万恶之源。

通过在EG(异常)中写入异常来生成异常，其中EG指的是执行程序全局变量。从C代码中抛出异常不涉及堆栈展开，而是通过返回值失败代码向上传播或检查EG(异常)。仅当控件重新进入虚拟机代码时才会实际处理该异常。

在某些情况下，几乎所有VM指令都可以直接或间接导致异常。例如，如果使用自定义错误处理程序，任何“未定义变量”通知都可能导致异常。我们希望避免检查在每个VM指令之后是否已设置EG(异常)。相反，使用了一个小技巧：

抛出异常时，当前执行数据的当前opline将替换为伪HANDLE_EXCEPTION opline(这显然不会修改op数组，它只会重定向指针)。发生异常的opline备份到EG(opline_before_exception)。

这意味着当控制返回到主虚拟机分派循环时，将调用HANDLE_EXCEPTION操作码。这个方案有一个小问题：它要求a)存储在执行数据中的opline实际上是当前执行的opline(否则opline_before_exception将是错误的)和b)虚拟机使用来自执行数据的opline继续执行(否则不会调用HANDLE_EXCEPTION)。

虽然这些要求可能听起来微不足道，但事实并非如此。原因是虚拟机可能正在处理与执行数据中存储的opline不同步的不同opline变量。在PHP 7之前，这只发生在很少使用的GOTO和SWITCH虚拟机中，而在PHP 7中，这实际上是默认的操作模式：如果编译器支持它，则opline存储在全局寄存器中。

因此，在执行可能抛出的任何操作之前，必须将本地opline写回执行数据(SAVE_OPLINE操作)。类似地，在任何潜在抛出操作之后，必须从执行数据(主要是CHECK_EXCEPTION操作)填充局部opline。

现在，这个机制是在抛出异常后导致HANDLE_EXCEPTION操作码执行的原因。但是它做了什么？首先，它确定是否在try块中抛出异常。为此，op数组包含一个try_catch_elements数组，用于跟踪try，catch和finally块的opline偏移量：
```
typedef struct _zend_try_catch_element {
	uint32_t try_op;
	uint32_t catch_op;  /* ketchup! */
	uint32_t finally_op;
	uint32_t finally_end;
} zend_try_catch_element;
```

现在我们假装最后的块不存在，因为它们是一个完全不同的兔子洞。 假设我们确实在try块中，VM需要清理在抛出opline之前开始的所有未完成的操作，并且不要跨越try块的末尾。

这涉及释放当前在飞行中的所有呼叫的堆栈帧和相关数据，以及释放临时的临时数据。 在大多数情况下，临时工具是短暂的，直到消费指令直接跟随生成指令。 但是，实时范围可能会发生多个可能抛出的指令：
```
# (array)[] + throwing()
L0:   T0 = CAST (array) []
L1:   INIT_FCALL (0 args) "throwing"
L2:   V1 = DO_FCALL
L3:   T2 = ADD T0, V1
```

在这种情况下，T0变量在指令L1和L2期间是活动的，因此如果函数调用抛出则需要销毁。 一种特殊的临时类型往往具有特别长的有效范围：循环变量。 例如：
```
# foreach ($array as $value) throw $ex;
L0:   V0 = FE_RESET_R $array, ->L4
L1:   FE_FETCH_R V0, $value, ->L4
L2:   THROW $ex
L3:   JMP ->L1
L4:   FE_FREE V0
```

这里“循环变量”V0从L1到L3(通常总是跨越整个循环体)。 实时范围使用以下结构存储在op数组中：
```
typedef struct _zend_live_range {
    uint32_t var; /* low bits are used for variable type (ZEND_LIVE_* macros) */
    uint32_t start;
    uint32_t end;
} zend_live_range;
```

这里var是范围适用的(操作数编码)变量，start是开始opline偏移(不包括生成指令)，而end是结束opline偏移(包括消耗指令)。当然，只有在不立即消耗临时值时才存储有效范围。

var的低位用于存储变量的类型，可以是以下之一：

* ZEND_LIVE_TMPVAR：这是一个“正常”变量。它拥有一个普通的zval值。释放此变量的行为类似于免费操作码。
* ZEND_LIVE_LOOP：这是一个foreach循环变量，它不仅仅包含一个简单的zval。这对应于FE_FREE操作码。
* ZEND_LIVE_SILENCE：用于实现错误抑制运算符。旧的错误报告级别将备份到临时并稍后还原。如果抛出异常，我们显然也希望恢复它。这对应于END_SILENCE。
* ZEND_LIVE_ROPE：这用于绳索串联，在这种情况下，临时是一个固定大小的zend_string *指针数组，它们位于堆栈上。在这种情况下，必须释放所有已填充的字符串。大致对应于END_ROPE。

在这种情况下要考虑的一个棘手的问题是，如果他们的生成或他们的消费指令抛出，临时工作是否应该被释放。请考虑以下简单代码：
```
T2 = ADD T0, T1
ASSIGN $v, T2
```

如果ADD抛出异常，T2临时是自动释放，还是ADD指令对此负责？ 同样，如果ASSIGN抛出，T2应该自动释放，还是ASSIGN必须自己处理？ 在后一种情况下，答案很明确：即使抛出异常，指令也总是负责释放其操作数。

结果操作数的情况比较棘手，因为这里的答案在PHP 7.1和7.2之间发生了变化：在PHP 7.1中，指令负责在异常情况下释放结果。 在PHP 7.2中，它会自动释放(并且指令负责确保始终填充结果)。 这种变化的动机是实现许多基本指令(如ADD)的方式。 他们通常的结构大致如下：
```
1. read input operands
2. perform operation, write it into result operand
3. free input operands (if necessary)
```

这是有问题的，因为PHP处于非常不幸的位置，不仅支持异常和析构函数，而且还支持抛出析构函数(这是编译器工程师惊恐地喊叫的地方)。 因此，步骤3可以抛出，此时结果已经填充。 为避免此边缘情况下的内存泄漏，释放结果操作数的责任已从指令转移到异常处理机制。

一旦我们执行了这些清理操作，我们就可以继续执行catch块了。 如果没有catch(并且没有最终)，我们将展开堆栈，即破坏当前堆栈帧并为父帧提供处理异常的镜头。

因此，您完全了解整个异常处理业务的丑陋程度，我将介绍与投掷析构函数相关的另一个小问题。 它在实践中并不具有远程相关性，但我们仍需要处理它以确保正确性。 考虑以下代码：
```
foreach (new Dtor as $value) {
    try {
        echo "Return";
        return;
    } catch (Exception $e) {
        echo "Catch";
    }
}
```

现在假设Dtor是一个带有抛出析构函数的Traversable类。 此代码将生成以下操作码序列，循环体缩进以提高可读性：
```
L0:   V0 = NEW 'Dtor', ->L2
L1:   DO_FCALL
L2:   V2 = FE_RESET_R V0, ->L11
L3:   FE_FETCH_R V2, $value
L4:       ECHO 'Return'
L5:       FE_FREE (free on return) V2   # <- return
L6:       RETURN null                   # <- return
L7:       JMP ->L10
L8:       CATCH 'Exception' $e
L9:       ECHO 'Catch'
L10:  JMP ->L3
L11:  FE_FREE V2                        # <- the duplicated instr
```

重要的是，请注意“return”被编译为循环变量的FE_FREE和RETURN。 现在，如果FE_FREE抛出会发生什么，因为Dtor有一个抛出析构函数？ 通常，我们会说这个指令在try块中，所以我们应该调用catch。 但是，此时循环变量已被破坏！ catch会丢弃异常，我们将尝试继续迭代已经死循环变量。

这个问题的原因是，虽然抛出FE_FREE在try块内，但它是L11中FE_FREE的副本。 逻辑上，这是异常“真正”发生的地方。 这就是为什么由break产生的FE_FREE被注释为FREE_ON_RETURN。 这指示异常处理机制将异常源移动到原始释放指令。 因此上面的代码不会运行catch块，而是会生成一个未捕获的异常。

##### 最后处理(Finally handling)

PHP与finally块的历史有点困扰。 PHP 5.5首先引入了finally块，或者更确切地说：finally块的真正错误实现。 PHP 5.6,7.0和7.1中的每一个都附带了最终实现的重大改写，每个都修复了一大堆错误，但并未完全达到完全正确的实现。看起来PHP 7.1终于成功了(手指交叉)。

在编写本节时，我惊讶地发现，从当前实现和我目前的理解的角度来看，最终处理实际上并不是那么复杂。实际上，在许多方面，通过不同的迭代实现变得更简单，而不是更复杂。这表明对问题的理解不足会导致实现过于复杂和错误(尽管，公平地说，PHP 5实现的复杂性直接源于缺少AST)。

最后，只要控制退出try块，通常(例如使用return)或异常(通过抛出)就会运行块。有几个有趣的边缘情况要考虑，我将在进入实现之前快速说明。考虑：
```
try {
    throw new Exception();
} finally {
    return 42;
}
```

怎么了？ 最后获胜并且函数返回42.考虑：
```
try {
    return 24;
} finally {
    return 42;
}
```

最后赢了，功能返回42.最后总是获胜。

PHP禁止跳出finally块。 例如，禁止以下内容：
```
foreach ($array as $value) {
    try {
        return 42;
    } finally {
        continue;
    }
}
```

上面的代码示例中的“continue”将生成编译错误。 重要的是要理解这种限制纯粹是装饰性的，可以通过使用“众所周知的”捕获控制委托模式轻松解决：
```
foreach ($array as $value) {
    try {
        try {
            return 42;
        } finally {
            throw new JumpException;
        }
    } catch (JumpException $e) {
        continue;
    }
}
```

存在的唯一真正限制是不可能跳转到最终块，例如 最后禁止从外面执行goto到最终内部的标签。

随着预赛的开始，我们可以看看最终如何运作。 该实现使用两个操作码，FAST_CALL和FAST_RET。 粗略地说，FAST_CALL用于跳转到finally块，而FAST_RET用于跳出它。 让我们考虑最简单的情况：
```
try {
    echo "try";
} finally {
    echo "finally";
}
echo "finished";
```

此代码编译为以下操作码序列：
```
L0:   ECHO string("try")
L1:   T0 = FAST_CALL ->L3
L2:   JMP ->L5
L3:   ECHO string("finally")
L4:   FAST_RET T0
L5:   ECHO string("finished")
L6:   RETURN int(1)
```

FAST_CALL将自己的位置存储到T0中并跳转到L3的finally块。 当达到FAST_RET时，它会跳回到存储在T0中的位置(之后)。 在这种情况下，这将是L2，这只是在finally块周围跳转。 这是没有特殊控制流(返回或异常)的基本情况。 我们现在考虑一下例外情况：
```
try {
    throw new Exception("try");
} catch (Exception $e) {
    throw new Exception("catch");
} finally {
    throw new Exception("finally");
}
```

处理异常时，我们必须考虑抛出的异常相对于最近的try / catch / finally块的位置：

1. 抛出try，匹配catch：填充$ e并跳转到catch。
2. 抛出catch或尝试没有匹配的catch，如果有一个finally块：跳转到finally块，这次将异常备份到FAST_CALL临时(而不是在那里存储返回地址。)
3. 从最后抛出：如果FAST_CALL临时中存在备份异常，则将其链接为抛出异常的先前异常。继续将异常冒泡到下一个try / catch / finally。
4. 否则：继续将异常冒泡到下一个try / catch / finally。

在这个例子中，我们将完成前三个步骤：首先尝试抛出，触发跳转到catch。 Catch也会抛出，触发跳转到finally块，并在FAST_CALL临时中备份异常。然后finally块也抛出，因此“finally”异常将冒出“catch”异常集作为其先前的异常。

上一个示例的一个小变化是以下代码：
```
try {
    try {
        throw new Exception("try");
    } finally {}
} catch (Exception $e) {
    try {
        throw new Exception("catch");
    } finally {}
} finally {
    try {
        throw new Exception("finally");
    } finally {}
}
```

此处的所有内部最终块都是异常输入，但通常保留(通过FAST_RET)。 在这种情况下，从父try / catch / finally块开始恢复先前描述的异常处理过程。 这个父try / catch存储在FAST_RET操作码中(这里是“try-catch(0)”)。

这基本上涵盖了最终和例外的相互作用。 但最终回归怎么样？
```
try {
    throw new Exception("try");
} finally {
    return 42;
}
```

操作码序列的相关部分是这样的：
```
L4:   T0 = FAST_CALL ->L6
L5:   JMP ->L9
L6:   DISCARD_EXCEPTION T0
L7:   RETURN 42
L8:   FAST_RET T0
```

额外的DISCARD_EXCEPTION操作码负责丢弃try块中抛出的异常(请记住：最后获胜中的返回)。 尝试回归怎么样？
```
try {
    $a = 42;
    return $a;
} finally {
    ++$a;
}
```

此处的例外返回值是42，而不是43.返回值由返回$ a行确定，$ a的任何进一步修改都无关紧要。 代码导致：
```
L0:   ASSIGN $a, 42
L1:   T3 = QM_ASSIGN $a
L2:   T1 = FAST_CALL ->L6, T3
L3:   RETURN T3
L4:   T1 = FAST_CALL ->L6      # unreachable
L5:   JMP ->L8                 # unreachable
L6:   PRE_INC $a
L7:   FAST_RET T1
L8:   RETURN null
```

其中两个操作码无法访问，因为它们在返回后直接发生。 这些将在优化期间被删除，但我在这里显示未经优化的操作码。 这里有两个有趣的事情：首先，使用QM_ASSIGN(基本上是“复制到临时”指令)将$ a复制到T3中。 这可以防止后来修改$ a影响返回值。 其次，T3也传递给FAST_CALL，后者将备份T1中的值。 如果稍后丢弃try块的返回(例如，因为最终抛出或返回)，则该机制将用于释放未使用的返回值。

所有这些单独的机制都很简单，但在组成时需要注意一些。 考虑以下示例，其中Dtor再次是一些带有抛出析构函数的Traversable类：
```
try {
    foreach (new Dtor as $v) {
        try {
            return 1;
        } finally {
            return 2;
        }
    }
} finally {
    echo "finally";
}
```

此代码将生成以下操作码：
```
L0:   V2 = NEW (0 args) "Dtor"
L1:   DO_FCALL
L2:   V4 = FE_RESET_R V2 ->L16
L3:   FE_FETCH_R V4 $v ->L16
L4:       T5 = FAST_CALL ->L10         # inner try
L5:       FE_FREE (free on return) V4
L6:       T1 = FAST_CALL ->L19
L7:       RETURN 1
L8:       T5 = FAST_CALL ->L10         # unreachable
L9:       JMP ->L15
L10:      DISCARD_EXCEPTION T5         # inner finally
L11:      FE_FREE (free on return) V4
L12:      T1 = FAST_CALL ->L19
L13:      RETURN 2
L14:      FAST_RET T5 try-catch(0)
L15:  JMP ->L3
L16:  FE_FREE V4
L17:  T1 = FAST_CALL ->L19
L18:  JMP ->L21
L19:  ECHO "finally"                   # outer finally
L20:  FAST_RET T1
```

第一次返回的序列(来自内部尝试)是FAST_CALL L10，FE_FREE V4，FAST_CALL L19，RETURN。 这将首先调用内部finally块，然后释放foreach循环变量，然后调用外部finally块然后返回。 第二次返回的序列(最终来自内部)是DISCARD_EXCEPTION T5，FE_FREE V4，FAST_CALL L19。 这首先丢弃内部try块的异常(或这里：返回值)，然后释放foreach循环变量，最后调用外部finally块。 请注意，在这两种情况下，这些指令的顺序与源代码中相关块的顺序相反。

##### 生成器(Generators)

生成器功能可能会暂停和恢复，因此需要特殊的VM堆栈管理。 这是一个简单的发电机：
```
function gen($x) {
    foo(yield $x);
}
```

这会产生以下操作码：
```
$x = RECV 1
GENERATOR_CREATE
INIT_FCALL_BY_NAME (1 args) string("foo")
V1 = YIELD $x
SEND_VAR_NO_REF_EX V1 1
DO_FCALL_BY_NAME
GENERATOR_RETURN null
```

在达到GENERATOR_CREATE之前，这将在普通VM堆栈上作为普通函数执行。然后GENERATOR_CREATE创建一个Generator对象，以及一个堆分配的execute_data结构(包括变量和参数的插槽，像往常一样)，VM堆栈上的execute_data被复制到该结构中。

当生成器再次恢复时，执行程序将使用堆分配的execute_data，但将继续使用主VM堆栈来推送调用帧。一个明显的问题是，正如前面的例子所示，可以在呼叫进行时中断发生器。这里，YIELD在调用foo()的调用帧已被推送到VM堆栈的位置执行。

这种相对不常见的情况是通过在产生控制时将活动调用帧复制到生成器结构中，并在恢复生成器时恢复它们来处理的。

从PHP 7.1开始使用此设计。以前，每个生成器都有自己的4KiB VM页面，当生成器恢复时，它将被交换到执行程序中。这避免了复制调用帧的需要，但增加了内存使用量。

##### 智能分支(Smart branches)

比较指令直接跟随条件跳转是很常见的。 例如：
```
L0:   T2 = IS_EQUAL $a, $b
L1:   JMPZ T2 ->L3
L2:   ECHO "equal"
```

由于此模式非常常见，因此所有比较操作码(例如IS_EQUAL)都实现了智能分支机制：它们检查下一条指令是否为JMPZ或JMPNZ指令，如果是，则自行执行相应的跳转操作。

智能分支机制仅检查下一条指令是否是JMPZ / JMPNZ，但实际上并不检查其操作数是否实际上是比较的结果，或其他内容。 在比较和后续跳转不相关的情况下，这需要特别小心。 例如，代码($ a == $ b)+($ d？$ e：$ f)生成：
```
L0:   T5 = IS_EQUAL $a, $b
L1:   NOP
L2:   JMPZ $d ->L5
L3:   T6 = QM_ASSIGN $e
L4:   JMP ->L6
L5:   T6 = QM_ASSIGN $f
L6:   T7 = ADD T5 T6
L7:   FREE T7
```

请注意，在IS_EQUAL和JMPZ之间插入了NOP。 如果此NOP不存在，则分支将最终使用IS_EQUAL结果，而不是JMPZ操作数。

##### 运行时缓存(Runtime cache)

因为操作码数组在多个进程之间是共享的(没有锁)，所以它们是严格不可变的。但是，运行时值可以缓存在单独的“运行时缓存”中，该缓存基本上是指针数组。文字可以具有关联的运行时缓存条目(或多个)，其存储在其u2槽中。

运行时缓存条目有两种类型：第一种是普通缓存条目，例如INIT_FCALL使用的条目。在INIT_FCALL查找被调用函数一次(基于其名称)之后，函数指针将被缓存在关联的运行时缓存槽中。

第二种类型是多态缓存条目，它们只是两个连续的缓存槽，其中第一个存储类条目，第二个存储实际数据。它们用于FETCH_OBJ_R之类的操作，其中高速缓存某个类的属性表中属性的偏移量。如果下一次访问发生在同一个类(很可能)，将使用缓存的值。否则，执行更昂贵的查找操作，并为新的类条目缓存结果。

##### 虚拟机中断(VM interrupts)

在PHP 7.0之前，执行超时用于由longjump直接从信号处理程序处理到关闭序列。 正如您可能想象的那样，这引起了各种不愉快。 由于PHP 7.0超时被延迟，直到控制返回到虚拟机。 如果它在某个宽限期内没有返回，则该过程将中止。 由于PHP 7.1 pcntl信号处理程序使用与执行超时相同的机制。

当信号挂起时，将设置VM中断标志，虚拟机会在某些点检查此标志。 不会在每条指令执行检查，而是仅在跳转和调用时执行检查。 因此，在返回VM时不会立即处理中断，而是在线性控制流的当前部分结束时处理。

##### 专业化(Specialization)

如果你看一下[VM定义](https://github.com/php/php-src/blob/master/Zend/zend_vm_def.h)文件，你会发现操作码处理程序定义如下：
```
ZEND_VM_HANDLER(1, ZEND_ADD, CONST|TMPVAR|CV, CONST|TMPVAR|CV)
```

这里的1是操作码编号，ZEND_ADD是其名称，而另外两个参数指定了指令接受的操作数类型。[生成的虚拟机代码](https://github.com/php/php-src/blob/master/Zend/zend_vm_execute.h)(由[zend_vm_gen.php](https://github.com/php/php-src/blob/master/Zend/zend_vm_gen.php)生成)将包含每个可能的操作数类型组合的专用处理程序。它们的名称类似于ZEND_ADD_SPEC_CONST_CONST_HANDLER。

通过替换处理程序主体中的某些宏来生成专用处理程序。显而易见的是OP1_TYPE和OP2_TYPE，但GET_OP1_ZVAL_PTR()和FREE_OP1()等操作也是专用的。

ADD的处理程序指定它接受CONST | TMPVAR | CV操作数。这里的TMPVAR意味着操作码接受TMP和VAR，但要求这些不单独使用。请记住，对于大多数用途，TMP和VAR之间的唯一区别是后者可以包含引用。对于像ADD这样的操作码(无论如何，引用都在慢速路径上)，对此进行单独的专业化是不值得的。确实做出这种区分的其他操作码将在其操作数列表中使用TMP | VAR。

在基于操作数类型的特化之后，处理程序还可以专门处理其他因素，例如是否使用它们的返回值。ASSIGN_DIM专门基于以下OP_DATA操作码的操作数类型：
```
ZEND_VM_HANDLER(147, ZEND_ASSIGN_DIM,
    VAR|CV, CONST|TMPVAR|UNUSED|NEXT|CV, SPEC(OP_DATA=CONST|TMP|VAR|CV))
```

基于此签名，将生成2 * 4 * 4 = 32种不同的ASSIGN_DIM变体。 第二个操作数的规范还包含NEXT的条目。 这与特化无关，而是指定UNUSED操作数在此上下文中的含义：它表示这是一个追加操作($ arr [])。另一个例子：
```
ZEND_VM_HANDLER(23, ZEND_ASSIGN_ADD,
    VAR|UNUSED|THIS|CV, CONST|TMPVAR|UNUSED|NEXT|CV, DIM_OBJ, SPEC(DIM_OBJ))
```

这里我们知道第一个操作数是UNUSED意味着访问$this。 这是对象相关操作码的一般约定，例如FETCH_OBJ_R UNUSED，'prop'对应于$ this-> prop。 UNUSED第二个操作数再次暗示追加操作。 这里的第三个参数指定了extended_value操作数的含义：它包含一个区分$ a + = 1，$ a [$ b] + = 1和$ a-> b + = 1的标志。最后，SPEC(DIM_OBJ) )指示应为每个处理程序生成专门的处理程序。 (在这种情况下，将生成的处理程序总数非常重要，因为VM生成器知道某些组合是不可能的。例如，UNUSED op1仅与OBJ情况相关，等等)

最后，虚拟机生成器支持更复杂的专业化机制。 在定义文件的末尾，您将找到许多此形式的处理程序：
```
ZEND_VM_TYPE_SPEC_HANDLER(
    ZEND_ADD,
    (res_info == MAY_BE_LONG && op1_info == MAY_BE_LONG && op2_info == MAY_BE_LONG),
    ZEND_ADD_LONG_NO_OVERFLOW,
    CONST|TMPVARCV, CONST|TMPVARCV, SPEC(NO_CONST_CONST,COMMUTATIVE)
)
```

这些处理程序不仅基于VM操作数类型，还基于操作数在运行时可能采用的类型。 确定可能的操作数类型的机制是opcache优化基础结构的一部分，完全超出了本文的范围。 但是，假设有这样的信息，应该很清楚，这是一个添加形式int + int - > int的处理程序。 此外，SPEC注释告诉专业人员不应生成两个const操作数的变体，并且操作是可交换的，因此如果我们已经具有CONST + TMPVARCV特化，我们也不需要生成TMPVARCV + CONST。

##### Fast-path/slow-path split

许多操作码处理程序是使用快速路径/慢速路径拆分实现的，首先处理几个常见情况，然后再回到通用实现。 现在是我们查看一些实际代码的时候了，所以我将在这里粘贴整个SL(左移)实现：
```
ZEND_VM_HANDLER(6, ZEND_SL, CONST|TMPVAR|CV, CONST|TMPVAR|CV)
{
	USE_OPLINE
	zend_free_op free_op1, free_op2;
	zval *op1, *op2;

	op1 = GET_OP1_ZVAL_PTR_UNDEF(BP_VAR_R);
	op2 = GET_OP2_ZVAL_PTR_UNDEF(BP_VAR_R);
	if (EXPECTED(Z_TYPE_INFO_P(op1) == IS_LONG)
			&& EXPECTED(Z_TYPE_INFO_P(op2) == IS_LONG)
			&& EXPECTED((zend_ulong)Z_LVAL_P(op2) < SIZEOF_ZEND_LONG * 8)) {
		ZVAL_LONG(EX_VAR(opline->result.var), Z_LVAL_P(op1) << Z_LVAL_P(op2));
		ZEND_VM_NEXT_OPCODE();
	}

	SAVE_OPLINE();
	if (OP1_TYPE == IS_CV && UNEXPECTED(Z_TYPE_INFO_P(op1) == IS_UNDEF)) {
		op1 = GET_OP1_UNDEF_CV(op1, BP_VAR_R);
	}
	if (OP2_TYPE == IS_CV && UNEXPECTED(Z_TYPE_INFO_P(op2) == IS_UNDEF)) {
		op2 = GET_OP2_UNDEF_CV(op2, BP_VAR_R);
	}
	shift_left_function(EX_VAR(opline->result.var), op1, op2);
	FREE_OP1();
	FREE_OP2();
	ZEND_VM_NEXT_OPCODE_CHECK_EXCEPTION();
}
```

实现从在BP_VAR_R模式下使用GET_OPn_ZVAL_PTR_UNDEF获取操作数开始。这里的UNDEF部分意味着在CV情况下不执行未定义变量的检查，而是按原样返回UNDEF值。一旦我们有了操作数，我们检查两者是否都是整数，并且移位宽度在范围内，在这种情况下，结果可以直接计算，然后我们前进到下一个操作码。值得注意的是，此处的类型检查并不关心操作数是否为UNDEF，因此使用GET_OPn_ZVAL_PTR_UNDEF是合理的。

如果操作数不满足快速路径，我们将回退到以SAVE_OPLINE()开头的通用实现。这是我们“潜在投掷操作跟随”的信号。在继续之前，处理未定义变量的情况。在这种情况下，GET_OPn_UNDEF_CV将发出未定义的变量通知并返回NULL值。

接下来，调用泛型shift_left_function并将其结果写入EX_VAR(opline-> result.var)。最后，释放输入操作数(如果需要)，我们通过异常检查前进到下一个操作码(这意味着在前进之前重新加载opline)。

因此，这里的快速路径保存了对未定义变量的两个检查，对通用运算符函数的调用，操作数的释放，以及异常处理的opline的保存和重载。大多数性能敏感的操作码以类似的方式显示出来。

##### 虚拟机宏(VM macros)

从前面的代码清单中可以看出，虚拟机实现可以自由使用宏。 其中一些是普通的C宏，而其他一些是在生成虚拟机期间解决的。 特别是，这包括许多用于获取和释放指令操作数的宏：
```
OPn_TYPE
OP_DATA_TYPE

GET_OPn_ZVAL_PTR(BP_VAR_*)
GET_OPn_ZVAL_PTR_DEREF(BP_VAR_*)
GET_OPn_ZVAL_PTR_UNDEF(BP_VAR_*)
GET_OPn_ZVAL_PTR_PTR(BP_VAR_*)
GET_OPn_ZVAL_PTR_PTR_UNDEF(BP_VAR_*)
GET_OPn_OBJ_ZVAL_PTR(BP_VAR_*)
GET_OPn_OBJ_ZVAL_PTR_UNDEF(BP_VAR_*)
GET_OPn_OBJ_ZVAL_PTR_DEREF(BP_VAR_*)
GET_OPn_OBJ_ZVAL_PTR_PTR(BP_VAR_*)
GET_OPn_OBJ_ZVAL_PTR_PTR_UNDEF(BP_VAR_*)
GET_OP_DATA_ZVAL_PTR()
GET_OP_DATA_ZVAL_PTR_DEREF()

FREE_OPn()
FREE_OPn_IF_VAR()
FREE_OPn_VAR_PTR()
FREE_UNFETCHED_OPn()
FREE_OP_DATA()
FREE_UNFETCHED_OP_DATA()
```

如您所见，这里有很多变化。 BP_VAR_ *参数指定获取模式并支持与FETCH_ *指令相同的模式(FUNC_ARG除外)。

GET_OPn_ZVAL_PTR()是基本的操作数提取。它将在未定义的CV上发出通知，并且不会取消引用操作数。正如我们已经知道的那样，GET_OPn_ZVAL_PTR_UNDEF()是一个不检查未定义CV的变体。 GET_OPn_ZVAL_PTR_DEREF()包括zval的DEREF。这是专门的GET操作的一部分，因为仅对CV和VAR进行解除引用，但对于CONST和TMP则不需要。因为此宏需要区分TMP和VAR，所以它只能与TMP | VAR专门化(但不能用于TMPVAR)一起使用。

GET_OPn_OBJ_ZVAL_PTR *()变体另外处理UNUSED操作数的情况。如前所述，按照约定$ this访问使用UNUSED操作数，因此GET_OPn_OBJ_ZVAL_PTR *()宏将为UNUSED操作返回对EX(This)的引用。

最后，还有一些PTR_PTR变种。这里的命名是PHP的剩余5次，其中实际上使用了双向间接的zval指针。这些宏用于写操作，因此仅支持CV和VAR类型(其他任何东西都返回NULL)。它们与正常的PTR提取不同之处在于它们取消了VAR操作数。

然后使用FREE_OP *()宏释放获取的操作数。要进行操作，它们需要定义一个zend_free_op free_opN变量，GET操作将该值存储到free中。基线FREE_OPn()操作将释放TMP和VAR，但不释放免费的CV和CONST。 FREE_OPn_IF_VAR()完全按照它所说的做法：只有当它是VAR时才释放操作数。

FREE_OP * _VAR_PTR()变体与PTR_PTR提取一起使用。它只会释放VAR操作数，并且只有它们不是间接的。

FREE_UNFETCHED_OP *()变体用于必须在使用GET获取操作数之前释放操作数的情况。如果在操作数提取之前抛出异常，则通常会发生这种情况。

除了这些专门的宏之外，还有相当多的普通类型的宏。 VM定义了三个宏来控制运行操作码处理程序后发生的事情：
```
ZEND_VM_CONTINUE()
ZEND_VM_ENTER()
ZEND_VM_LEAVE()
ZEND_VM_RETURN()
```

CONTINUE将继续正常执行操作码，而ENTER和LEAVE用于进入/离开嵌套函数调用。 这些操作的具体细节取决于VM的编译方式(例如，是否使用全局寄存器，如果使用，则使用哪个)。 从广义上讲，这些将在继续之前将某些状态与全局变量同步。 RETURN用于实际退出主VM循环。

ZEND_VM_CONTINUE()期望opline事先更新。 当然，还有更多与之相关的宏：
|                                        | Continue? | Check exception? | Check interrupt? |
|:---------------------------------------|:---------:|:----------------:|:----------------:|
|ZEND_VM_NEXT_OPCODE()                   |   yes     |       no         |       no         |
|ZEND_VM_NEXT_OPCODE_CHECK_EXCEPTION()   |   yes     |       yes        |       no         |
|ZEND_VM_SET_NEXT_OPCODE(op)             |   no      |       no         |       no         |
|ZEND_VM_SET_OPCODE(op)                  |   no      |       no         |       yes        |
|ZEND_VM_SET_RELATIVE_OPCODE(op, offset) |   no      |       no         |       yes        |
|ZEND_VM_JMP(op)                         |   yes     |       yes        |       yes        |

该表显示宏是否包含隐式ZEND_VM_CONTINUE()，它是否将检查异常以及是否将检查VM中断。

除此之外，还有SAVE_OPLINE()，LOAD_OPLINE()和HANDLE_EXCEPTION()。 正如在异常处理一节中所提到的，SAVE_OPLINE()在操作码处理程序中的第一个潜在抛出操作之前使用。 如有必要，它会将VM(可能位于全局寄存器中)使用的opline写回执行数据。 LOAD_OPLINE()是相反的操作，但现在它看起来很少使用，因为它已经有效地转入ZEND_VM_NEXT_OPCODE_CHECK_EXCEPTION()和ZEND_VM_JMP()。

在您已经知道抛出异常之后，HANDLE_EXCEPTION()用于从操作码处理程序返回。 它执行LOAD_OPLINE和CONTINUE的组合，这将有效地分派给HANDLE_EXCEPTION操作码。

当然，还有更多的宏(总是有更多的宏...)，但这应该涵盖最重要的部分。
