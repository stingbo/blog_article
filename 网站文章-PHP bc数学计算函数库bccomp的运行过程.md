###### 背景

    很多平常时时接触的事物、现象，当你习以为常，司空见惯之后，往往不会深究，等到某刻突然有人问起时，只会一时语塞。

###### PHP函数库bcmath

文件位置：`源码目录/ext/bcmath`

> function : int bccomp ( string $left_operand , string $right_operand [, int $scale = int ] )

1. 比较两个参数的符号，正负符号不同，则直接可以判断出两个数的大小;相同继续向下运行.

2. 比较两个参数的长度(数量级)

#### 未完待续