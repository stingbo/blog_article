*注:*

在查看PHP7源码的时候看到一个`ZEND_PARSE_PARAMETERS_START()`与`ZEND_PARSE_PARAMETERS_END()`代码块，开始不太理解，找到官方RFC才明白。

文章主要内容翻译自PHP官方的RFC，欢迎指正[查看原文](https://wiki.php.net/rfc/fast_zpp)

##### 描述

PHP内部函数使用`zend_parse_parameters()`<abbr title="Application Programming Interface">API</abbr>接受参数，将输入参数转换成C变量，这个函数使用`scanf()`方法进行参数定义，所需数据的数量和类型由包含说明符列表的字符串定义（“s”-表示字符串，“l”表示长整型等），不幸的是，每次调用这个函数时都要对这个这个字符串进行解析，这会加重性能开销。

例如，在服务wordpress主页时，`zend_parse_parameters()`会占用CPU大约6％的运行时间。

对于一些非常简单的函数，如`is_string()`或`ord()`，`zend_parse_parameters()`的开销可能约为90％。

##### 提案

我们提出了另一个API(fast zpp)以快速解析参数，它应该被用于最有用的函数里。API是基于C语言的宏，这可以将优化后的代码直接嵌入到内部函数体中。

我们不建议删除现有的API，并建议仅对最常用的函数使用快速API以提高性能。

我将在下面的示例中解释API:
```
if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "al|zb", &input, &offset, &z_length, &preserve_keys) == FAILURE) {
	return;
}
```

```
ZEND_PARSE_PARAMETERS_START(2, 4)
	Z_PARAM_ARRAY(input)
	Z_PARAM_LONG(offset)
	Z_PARAM_OPTIONAL
	Z_PARAM_ZVAL(z_length)
	Z_PARAM_BOOL(preserve_keys)
ZEND_PARSE_PARAMETERS_END();
```

第一个代码片段只取自`PHP_FUNCTION(array_slice)`，第二个是使用新API替换后的。根据代码实际上可以望文生义。

* `ZEND_PARSE_PARAMETERS_START()`-接受两个参数，最少传入和最多传入的参数数量。
* `Z_PARAM_ARRAY()`-将下一个参数作为数组
* `Z_PARAM_LONG`-作为长整型参数
* `Z_PARAM_OPTIONAL`-此项说明剩下的参数是可选的。

新的API涵盖了现有API的所有可能性。下表列出了旧说明符和新宏之间的对应关系。

|specifier|Fast ZPP API macro|args|
|---|---|---|
|&#124;|Z_PARAM_OPTIONAL||
|a|Z_PARAM_ARRAY(dest)|dest - zval*|
|A|Z_PARAM_ARRAY_OR_OBJECT(dest)|dest - zval*|
|b|Z_PARAM_BOOL(dest)|dest - zend_bool|
|C|Z_PARAM_CLASS(dest)|dest - zend_class_entry*|
|d|Z_PARAM_DOUBLE(dest)|dest - double|
|f|Z_PARAM_FUNC(fci, fcc)|fci - zend_fcall_info, fcc - zend_fcall_info_cache|
|h|Z_PARAM_ARRAY_HT(dest)|dest - HashTable*|
|H|Z_PARAM_ARRAY_OR_OBJECT_HT(dest)|dest - HashTable*|
|l|Z_PARAM_LONG(dest)|dest - long|
|L|Z_PARAM_STRICT_LONG(dest)|dest - long|
|o|Z_PARAM_OBJECT(dest)|dest - zval*|
|O|Z_PARAM_OBJECT_OF_CLASS(dest, ce)|dest - zval*|
|p|Z_PARAM_PATH(dest, dest_len)|dest - char*, dest_len - int|
|P|Z_PARAM_PATH_STR(dest)|dest - zend_string*|
|r|Z_PARAM_RESOURCE(dest)|dest - zval*|
|s|Z_PARAM_STRING(dest, dest_len)|dest - char*, dest_len - int|
|S|Z_PARAM_STR(dest)|dest - zend_string*|
|z|Z_PARAM_ZVAL(dest)|dest - zval*|
| |Z_PARAM_ZVAL_DEREF(dest)|dest - zval*|
|+|Z_PARAM_VARIADIC('+', dest, num)|dest - zval*, num int|
|*|Z_PARAM_VARIADIC('*', dest, num)|dest - zval*, num int|

`!`的效果和关联修饰符可以使用相同宏的扩展版本来实现，例如`Z_PARAM_ZVAL_EX(dest, check_null, separate)`。
