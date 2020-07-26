### String

1、为什么String类设计为final

1、String类型不可变才能维护一个String常量池。当两个String变量拥有同样的值时，可以将变量直接指向String常量池。

2、String类型不可变，所以创建的时候就计算了hashcode且不会再变，是得Hashmap中可以使用String作为可靠的Key。

## Hashmap,ConcurrentHashMap