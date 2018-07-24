__注:__
转载文章翻译自Nikita的文章，欢迎指正[查看原文](http://nikic.github.io/2015/06/19/Internal-value-representation-in-PHP-7-part-2.html)

在第一部分中，讨论了PHP5和PHP7之间内部值表示的高级改变。注意，主要区别在于zvals不再单独分配，也不自行存储引用计数。整数或浮点数等简单值可以直接存储在zval中，而复杂数值则使用指向单独结构的指针表示。

复杂的zval值的额外结构使用公共头，它由`zend_refcounted`来定义(博主注：PHP7.2与7又不一样)：
```
struct _zend_refcounted {
    uint32_t refcount;
    union {
        struct {
            ZEND_ENDIAN_LOHI_3(
                zend_uchar    type,
                zend_uchar    flags,
                uint16_t      gc_info)
        } v;
        uint32_t type_info;
    } u;
};
```

此头部含有`refcount`，值类型`type`与循环集合信息`(gc_info)`，以及一个特定类型插槽的标志`flags`。

下面将讨论各个复杂类型的详细信息并与PHP5里的实现进行对比。其中一个复杂的类型就是引用，这些引用在前一部分里已经介绍过了。另一种没涉及的类型是资源，因为我觉得它没啥意思。

##### 字符串(Strings)

PHP7使用`zend_string`类型表示字符串，其定义如下：
```
struct _zend_string {
    zend_refcounted   gc;
    zend_ulong        h;        /* hash value */
    size_t            len;
    char              val[1];
};
```

除了refcounted标头之外，字符串还包含了哈希缓存`h`，长度`len`和值`val`。哈希缓存用于避免每次用于在哈希表中查找键时重新计算字符串的哈希值。首次使用时，它将初始化为（非零）哈希。

如果你不熟悉相当广泛的`dirty C hacks`(不懂什么是[dirty hacks，请点击?](https://blog.blianb.com/archives/3416))知识，那么`val`的定义可能看起来很奇怪：它被声明为一个带有单个元素的char数组 - 但我们确定要存储长于一个字符的字符串吗？这使用了一种称为“struct hack”的技术：数组只用一个元素声明，但是在创建`zend_string`时，我们将它分配给一个更大的字符串。我们将仍然可以通过`val`成员访问更大的字符串。

当然，这是技术上未定义的潜规则，因为我们最终会在单字符数组的末尾读取和写入，但是C编译器知道在执行此操作时不要搞乱代码。C99以“flexible array members(灵活的数组成员)”的形式明确支持这一点，但是感谢我们亲爱的微软朋友，在实际上使用C99时，没有人需要跨平台兼容性。

与使用普通C字符串相比，新字符串类型具有一些优势：首先，它直接嵌入字符串长度。这意味着不再需要在所有地方传递字符串的长度。其次，由于字符串具有refcounted标头，因此可以在不使用zvals的情况下在多个位置共享字符串。这对于共享哈希表键尤其重要。

新的字符串类型也有一个很大的缺点：虽然很容易从zend_string获取一个C字符串（仅使用str-> val），但是又不可能直接从C字符串中获取一个zend_string - 你需要实际复制字符串的值到新分配的zend_string中。在处理文字字符串（C源代码中出现的常量字符串）时，非常不方便。

字符串可以有许多标志（存储在GC标志字段中）：
```
#define IS_STR_PERSISTENT           (1<<0) /* allocated using malloc */
#define IS_STR_INTERNED             (1<<1) /* interned string */
#define IS_STR_PERMANENT            (1<<2) /* interned string surviving request boundary */
```

持久化字符串，使用普通的系统分配器而不是Zend内存管理器（ZMM），因此可以保持超过一个请求的时间。将已使用的分配器指定为标志允许我们在zval中透明地使用持久化字符串，而之前在PHP5中需要事先将副本复制到ZMM中。

Interned字符串是在请求结束之前不会被销毁的字符串，因此不需要使用refcounting。它们也进行了重复数据删除，因此如果创建了新的interned字符串，引擎将先检查此内容的interned字符串是否已存在。字面上出现在PHP源代码中的所有字符串（包括字符串文字，变量和函数名称等）通常都是实例化的。永久字符串是在请求开始之前创建的实例化字符串。虽然正常的实例化字符串在请求关闭时被销毁，但永久字符串仍保持活动状态。

如果使用了opcache，则interned字符串将存储在共享内存（SHM）中，并在所有PHP工作进程中共享。在这种情况下，永久字符串的概念变得无关紧要，因为interned字符串永远不会被销毁。

##### 数组(Arrays)

我不会在这里讨论新数组类型中实现的细节，因为这已在以前[这篇文章](https://nikic.github.io/2014/12/22/PHPs-new-hashtable-implementation.html)中介绍过。由于最近的变化，它在某些细节上已不再准确，但所有概念仍然相同。

我在这里只提一个与数组相关的新概念，因为它没有在哈希表帖子中介绍：不可变数组。这些本质上是值相等的实例化字符串，因为它们不使用引用计数，并且一直存在到请求结束（或更久）。

由于某些内存管理问题，仅在启用opcache时才使用不可变数组。要了解这可以产生什么样的差异，请参考以下脚本：
```
for ($i = 0; $i < 1000000; ++$i) {
    $array[] = ['foo'];
}
var_dump(memory_get_usage());
```

使用opcache时，内存使用量为32MB，但没有opcache时使用量会增加到390MB，因为在这种情况下，$array的每个元素都将获得['foo']的新副本。在这里完成实际复制（而不是引用refcount）的原因是文字虚拟操作数不使用引用计数来避免SHM损坏。我希望我们能够改善这个目前的灾难性案例，以便将来在没有opcache的情况下也能更好地运行。

##### PHP5 里的对象(Objects in PHP5)

在参考PHP7中的对象实现之前，让我们先了解PHP5中的工作原理并强调一些低效的问题：zval本身用于存储`zend_object_value`，其定义如下：
```
typedef struct _zend_object_value {
    zend_object_handle handle;
    const zend_object_handlers *handlers;
} zend_object_value;
```

句柄(handle)是对象的唯一ID，可用于查找对象数据。`handlers`是实现对象的各种行为的VTable函数指针。对于“普通”PHP对象，此`handle`表始终相同，但PHP扩展创建的对象可以使用一组自定义`handlers`来修改其行为方式（例如，通过重载）。

对象句柄用作“对象存储”的索引，“对象存储”是一个对象存储桶数组，定义如下：
```
typedef struct _zend_object_store_bucket {
    zend_bool destructor_called;
    zend_bool valid;
    zend_uchar apply_count;
    union _store_bucket {
        struct _store_object {
            void *object;
            zend_objects_store_dtor_t dtor;
            zend_objects_free_object_storage_t free_storage;
            zend_objects_store_clone_t clone;
            const zend_object_handlers *handlers;
            zend_uint refcount;
            gc_root_buffer *buffered;
        } obj;
        struct {
            int next;
        } free_list;
    } bucket;
} zend_object_store_bucket;
```

这里发生了很多事情。前三个成员只是一些元数据（是否调用了对象的析构函数，是否完全使用此存储桶以及某些递归算法访问此对象的次数）。下面的联合体区分了当前使用存储桶的情况或它是否是存储桶空闲列表的一部分。这个示例的重点是`struct _store_object`在哪被使用。

第一个成员对象是指向实际对象的指针（最终）。它没有直接嵌入到对象存储桶中，因为对象没有固定的大小。对象指针后面跟着三个处理销毁，释放和克隆的句柄。请注意，在PHP中，销毁和释放对象是不同的步骤，在某些情况下会跳过前者（“unclean shutdown”）。实际上克隆句柄几乎不会被用到。因为这些存储句柄不是普通对象句柄的一部分（无论出于何种原因），所以它们将为每个对象复制对象，而不是共享。

这些对象存储处理程序后跟一个指向普通对象处理程序的指针。如果在未知zval的情况下销毁对象，则这些对象会被存储起来（通常存储handlers）。

该存储桶还包含一个引用计数，考虑到在PHP5中zval已经存储了引用计数，这就有点奇怪了。为什么要弄两个呢？原因在于，虽然通常仅通过增加其引用来“复制”zval，但也存在发生硬拷贝的情况，即使用相同的`zend_object_value`分配全新的`zval`。在这种情况下，两个不同的zval最终使用相同的对象存储桶，因此它也需要重新计算。这种“双重引用计数”是PHP5 zval实现的固有问题之一。出于类似的原因，GC根缓冲区的缓冲指针也会复制。

现在让我们看一下对象存储所指向的实际对象。对于普通用户端对象，它定义如下：
```
typedef struct _zend_object {
    zend_class_entry *ce;
    HashTable *properties;
    zval **properties_table;
    HashTable *guards;
} zend_object;
```

`zend_class_entry`是指向此对象实例的类的指针。下面两个成员是用于存储对象属性的两种不同方式。对于动态属性（即在运行时添加但未在类中声明的属性），使用属性哈希表，它只是属性的键值映射（混淆）而已。

但是对于声明的属性，使用了优化：在编译期间，为每个此类属性分配一个索引，并将该属性的值存储在`properties_table`中的该索引处。属性名称及其索引之间的映射存储在类实体的哈希表中。因此，对于各个对象，避免了哈希表的内存开销。此外，属性的索引在运行时以多态方式缓存。

`guards`哈希表用于实现像`__get`这样的魔术方法的递归行为，在这暂不讨论。

除了之前提到的双重引用计数问题之外，对单属性的小对象（不计算zval），内存使用占136个字节也太严重。此外还有很多其它问题，例如，要获取对象zval上的属性，首先必须获取对象存储桶，然后是zend对象，然后是属性表，再然后是它指向的zval。因此，至少已经有四个间接层次（实际上不少于七个）。

##### PHP7 里的对象(Objects in PHP7)

PHP7试图通过消除双重引用计数，减少一些内存膨胀和减少间接性来改进这些问题。下面是新的`zend_object`结构：

```
struct _zend_object {
    zend_refcounted   gc;
    uint32_t          handle;
    zend_class_entry *ce;
    const zend_object_handlers *handlers;
    HashTable        *properties;
    zval              properties_table[1];
};
```

请注意，此结构现在（几乎）是对象剩下的所有内容：`zend_object_value`已被替换为指向对象和对象存储的直接指针，虽然没有完全消失，但也不太重要了。

除了现在包括惯常的`zend_refcounted`标头之外，您还可以看到对象值的句柄`(handle)`和处理程序`(handlers)`已移动到`zend_object`中。此外，`properties_table`现在也使用"struct hack"，因此`zend_object`和属性表将分配在一个块中。当然，属性表现在直接嵌入zvals，而不是包含指向它们的指针。

`guards`表不再直接存在于对象结构中。取而代之的是，如果需要，比如对象使用`__get`等，它将被存储在第一个`properties_table`槽中。但是如果不使用这些魔术方法，则省略了`guards`表。

先前存储在对象存储桶中的`dtor`，`free_storage`和`clone`处理程序现已移动到处理程序表中，结构如下：
```
struct _zend_object_handlers {
    /* offset of real object header (usually zero) */
    int                                     offset;
    /* general object functions */
    zend_object_free_obj_t                  free_obj;
    zend_object_dtor_obj_t                  dtor_obj;
    zend_object_clone_obj_t                 clone_obj;
    /* individual object functions */
    // ... rest is about the same in PHP 5
};
```

在处理程序表的顶部是一个偏移成员，它显然不是一个句柄。此偏移量与内部对象的表示方式有关：内部对象始终嵌入标准`zend_object`，但通常还会添加许多其他成员。在PHP5中，是通过在标准对象之后添加它们来完成的：
```
struct custom_object {
    zend_object std;
    uint32_t something;
    // ...
};
```

这意味着如果你得到一个`zend_object*`，你可以简单地将它转换为你自定义的`struct custom_object*`。这是在C中实现结构继承的标准方法。但是在PHP7中这种特殊方法存在问题：因为`zend_object`使用`struct hack`来存储属性表，PHP存储的属性将超过`zend_object`尾部，从而覆盖其他内部成员。这就是为什么在PHP7中，其他成员存储在标准对象之前：
```
struct custom_object {
    uint32_t something;
    // ...
    zend_object std;
};
```

但是，这意味着不再可能使用简单的强制转换直接在`zend_object*`和`struct custom_object*`之间进行转换，因为两者都由偏移量分隔。此偏移量是存储在对象处理程序表的第一个成员中的。在编译时，可以使用`offsetof()`宏来确定偏移量。

你可能想知道为什么PHP7对象仍然包含句柄。毕竟，我们现在直接存储一个指向`zend_object`的指针，因此我们不再需要用句柄来查找对象存储中的对象。

然而，仍然必需句柄，因为对象存储仍然存在，尽管形式明显减少。它现在是一个指向对象的简单数组。创建对象时，指向它的指针会在句柄索引处插入对象存储区，并在释放对象后删除。

为什么我们仍然需要对象存储？原因是，在请求关闭期间，运行用户端代码不安全，因为执行程序已经部分关闭。为避免这种情况，PHP将在关闭期间的开始运行所有对象的析构函数，并防止它们在以后的某个时间点运行。为此需要所有活动对象的列表。

此外，句柄对于调试很有用，因为它为每个对象提供了唯一的ID，因此很容易看出两个对象是否真的相同或只是具有一些相同的内容。尽管没有对象存储的概念，HHVM仍然存储对象句柄。

与PHP5实现相比，我们现在只有一个refcount（因为zval本身不再有）并且内存使用量要小得多：我们需要40个字节用于基础对象，16个字节用于每个声明的属性，而且已经包括它zval的了。间接使用量也显著减少，因为许多中间结构被丢弃或嵌入。因此，读取性情现在只是一个一级的连接，而不是四个。

##### Indirect zvals

此时我们已经涵盖了所有普通的zval类型，但是有一些特殊类型仅在某些情况下使用。PHP7中新添加了一个zval--`IS_INDIRECT`。

间接zval表示其值存储在某个其他位置。注意这与`IS_REFERENCE`类型的不同之处在于它直接指向另一个zval，而不是嵌入zval的`zend_reference`结构。

为了理解在什么情况下这种方式可能是必要的，请参考PHP变量的实现（尽管同样适用于对象属性存储）。

编译时已知的所有变量都被赋予索引，它们的值将存储在编译变量(compiled variables CV)表中的该索引处。然而，PHP还允许你通过使用可变变量来动态引用变量，或者如果你在全局范围内，则通过`$GLOBALS`动态引用变量。如果发生这样的访问，PHP将为函数或者脚本创建一个符号表，其中包含健值映射。

这导致了一个问题：如何同时支持两种形式的访问？我们需要基于表的CV访问以进行常规变量提取和基于symtable的varvars访问。在PHP5中，CV表使用了双向间接的`zval**`指针。通常这些指针指向实际的zvals的第二个`zval*`指针表：
```
+------ CV_ptr_ptr[0]
| +---- CV_ptr_ptr[1]
| | +-- CV_ptr_ptr[2]
| | |
| | +-> CV_ptr[0] --> some zval
| +---> CV_ptr[1] --> some zval
+-----> CV_ptr[2] --> some zval
```

现在，一旦符号表开始使用，第二个包含单个`zval*`指针表就不再使用了，并且`zval**`指针被更新为指向哈希表桶。这里假设三个变量分别为`$a`，`$b`和`$c`：
```
CV_ptr_ptr[0] --> SymbolTable["a"].pDataPtr --> some zval
CV_ptr_ptr[1] --> SymbolTable["b"].pDataPtr --> some zval
CV_ptr_ptr[2] --> SymbolTable["c"].pDataPtr --> some zval
```

在PHP7中，已经不可能使用相同的途径，因为当调整哈希表的大小时，指向哈希表桶的指针将失效。另外，PHP7使用了反向策略：对于存储在CV表中的变量，符号哈希表将包含指向CV条目的`INDIRECT`条目。CV表不会在符号表的生命周期内重新分配，所以无效指针就没有问题。

因此，如果你有一个包含CV变量`$a`，`$b`和`$c`以及一个动态创建的变量`$d`的函数，符号表可能看起来像这样：
```
SymbolTable["a"].value = INDIRECT --> CV[0] = LONG 42
SymbolTable["b"].value = INDIRECT --> CV[1] = DOUBLE 42.0
SymbolTable["c"].value = INDIRECT --> CV[2] = STRING --> zend_string("42")
SymbolTable["d"].value = ARRAY --> zend_array([4, 2])
```

间接zvals也可以指向`IS_UNDEF`zval，在这种情况下，它被视为哈希表不包含它关联的键。因此，如果`unset($a)`会把`UNDEF`类型写入`CV[0]`，然后这就被看作符号表不再具有键“a”了。

##### Constands and ASTs

在PHP5和PHP7中还有两种特殊类型`IS_CONSTANT`和`IS_CONSTANT_AST`值得一提。要了解这些操作，请参考以下示例：
```
function test($a = ANSWER,
              $b = ANSWER * ANSWER) {
    return $a + $b;
}

define('ANSWER', 42);
var_dump(test()); // int(42 + 42 * 42)
```

函数`test()`的参数的默认值是常量`ANSWER`-但是在声明函数时尚未定义此常量。只有在`define()`定义后，该常量才能使用。

因此，参数和属性默认值，常量和接受“静态表达式”的所有其他内容都能够推迟表达式的检测，直到首次使用的时候。

如果该值是常量（或类常量），这是后期检测的最常见情况，则使用具有常量名称的`IS_CONSTANT`zval来发信号通知。如果值是表达式，则使用指向抽象语法树（abstract syntax tree AST）的`IS_CONSTANT_AST`zval。
