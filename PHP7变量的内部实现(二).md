在第一部分中，讨论了PHP 5和PHP 7之间内部值表示的高级更改。 注意，主要区别在于zvals不再单独分配，也不自行存储引用计数。 整数或浮点数等简单值可以直接存储在zval中，而复杂数值则使用指向单独结构的指针表示。

复杂的zval值的额外结构使用公共头，它由`zend_refcounted`来定义(博主注：PHP 7.2与7又不一样)：
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

此头部含有`refcount`，值类型`type`与循环集合信息(`gc_info`)，以及一个特定类型插槽的标志`flags`。

下面将讨论各个复杂类型的详细信息并与PHP 5里实现进行对比。其中一个复杂的类型就是引用，这些引用在前一部分里已经介绍过了。另一种没涉及的类型是资源，因为我认为它没什么意思。

##### 字符串(Strings)

PHP 7使用`zend_string`类型表示字符串，其定义如下：
```
struct _zend_string {
    zend_refcounted   gc;
    zend_ulong        h;        /* hash value */
    size_t            len;
    char              val[1];
};
```

除了refcounted标头之外，字符串还包含了哈希缓存`h`，长度`len`和值`val`。哈希缓存用于避免每次用于在哈希表中查找键时重新计算字符串的哈希值。首次使用时，它将初始化为（非零）哈希。

如果你不熟悉相当广泛的`dirty C hacks`(不懂什么是[dirty hacks]()?)知识，那么`val`的定义可能看起来很奇怪：它被声明为一个带有单个元素的char数组 - 但我们确定要存储长于一个字符的字符串吗？这使用了一种称为“struct hack”的技术：数组只用一个元素声明，但是在创建`zend_string`时，我们将它分配给一个更大的字符串。我们将仍然可以通过`val`成员访问更大的字符串。

当然，这是技术上未定义的行为，因为我们最终会在单字符数组的末尾读取和写入，但是C编译器知道在执行此操作时不要搞乱代码。C99以“flexible array members(灵活的数组成员)”的形式明确支持这一点，但是感谢我们亲爱的微软朋友，在实际上使用C99时， 没有人需要跨平台兼容性。

与使用普通C字符串相比，新字符串类型具有一些优势：首先，它直接嵌入字符串长度。这意味着不再需要在所有地方传递字符串的长度。其次，由于字符串现在具有refcounted标头，因此可以在不使用zvals的情况下在多个位置共享字符串。这对于共享哈希表键尤其重要。

新的字符串类型也有一个很大的缺点：虽然很容易从zend_string获取一个C字符串（仅使用str-> val），但是不可能直接从C字符串中获取一个zend_string - 你需要实际复制字符串的值到新分配的zend_string中。在处理文字字符串（C源代码中出现的常量字符串）时，非常不方便。

字符串可以有许多标志（存储在GC标志字段中）：
```
#define IS_STR_PERSISTENT           (1<<0) /* allocated using malloc */
#define IS_STR_INTERNED             (1<<1) /* interned string */
#define IS_STR_PERMANENT            (1<<2) /* interned string surviving request boundary */
```

持久化字符串，使用普通的系统分配器而不是Zend内存管理器（ZMM），因此可以保持超过一个请求的时间。将已使用的分配器指定为标志允许我们在zval中透明地使用持久化字符串，而之前在PHP 5中需要事先将副本复制到ZMM中。

Interned字符串是在请求结束之前不会被销毁的字符串，因此不需要使用refcounting。它们也进行了重复数据删除，因此如果创建了新的interned字符串，引擎将先检查此内容的interned字符串是否已存在。字面上出现在PHP源代码中的所有字符串（包括字符串文字，变量和函数名称等）通常都是实例化的。永久字符串是在请求开始之前创建的实例化字符串。虽然正常的实例化字符串在请求关闭时被销毁，但永久字符串仍保持活动状态。

如果使用了opcache，则interned字符串将存储在共享内存（SHM）中，并在所有PHP工作进程中共享。在这种情况下，永久字符串的概念变得无关紧要，因为interned字符串永远不会被销毁。

##### 数组(Arrays)

我不会在这里讨论新数组实现的细节，因为这已在以前[这篇文章](https://nikic.github.io/2014/12/22/PHPs-new-hashtable-implementation.html)中介绍过。由于最近的变化，它在某些细节上已不再准确，但所有概念仍然相同。

我在这里只提一个与数组相关的新概念，因为它没有在哈希表帖子中介绍：不可变数组。这些本质上是值相等的实例化字符串，因为它们不使用引用计数，并且一直存在到请求结束（或更久）。

由于某些内存管理问题，仅在启用opcache时才使用不可变数组。要了解这可以产生什么样的差异，请考虑以下脚本：
```
for ($i = 0; $i < 1000000; ++$i) {
    $array[] = ['foo'];
}
var_dump(memory_get_usage());
```

使用opcache，内存使用量为32MiB，但没有opcache使用率会增加到390MB，因为在这种情况下，$array的每个元素都将获得['foo']的新副本。在这里完成实际复制（而不是引用refcount）的原因是文字虚拟操作数不使用引用计数来避免SHM损坏。我希望我们能够改善这个目前的灾难性案例，以便将来在没有opcache的情况下更好地工作。

##### PHP 5 里的对象(Objects in PHP 5)

在考虑PHP7中的对象实现之前，让我们首先了解PHP5中的工作原理并强调一些低效的问题：zval本身用于存储`zend_object_value`，其定义如下：
```
typedef struct _zend_object_value {
    zend_object_handle handle;
    const zend_object_handlers *handlers;
} zend_object_value;
```

句柄(handle)是对象的唯一ID，可用于查找对象数据。处理程序是实现对象的各种行为的VTable函数指针。对于“普通”PHP对象，此处理程序表将始终相同，但PHP扩展创建的对象可以使用一组自定义处理程序来修改其行为方式（例如，通过重载运算符）。

对象句柄用作“对象存储”的索引，“对象存储”是一个对象存储桶数组，定义如下：
