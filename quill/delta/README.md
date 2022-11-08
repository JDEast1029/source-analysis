# Delta源码分析

Delta能够用来描述内容以及内容的变化，可以用来描述任何富文本内容

##### AttributeMap内部的方法

- `compose(a, b, keepNull)` 合并a,b，是否保留空值

- `diff(a, b)` 获取a,b之间不同值的属性

- `invert(attr, base)` 如果base内key为undefined & attr中不为undefined，将key的值设为null

- `transform(a, b, priority)` a中key值为空取b的key值，如果priority为true，直接用b

#### OP

操作接口，包含`insert` `delete` `retain` `attributes` , 提供`length` 方法获取这次操作的长度

##### OpIterator

Op的遍历器
