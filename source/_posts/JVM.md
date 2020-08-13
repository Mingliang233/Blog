---
title: JVM
date: 2020/4/30 20:46:25
updated: 2020/7/27 23:00:25
categories:
- 后端工程师
tags:
- Java
- JVM
---

#### class文件信息

```c
ClassFile {
    u4             magic;
    u2             minor_version;
    u2             major_version;
    u2             constant_pool_count;
    cp_info        constant_pool[constant_pool_count-1];
    u2             access_flags;
    u2             this_class;
    u2             super_class;
    u2             interfaces_count;
    u2             interfaces[interfaces_count];
    u2             fields_count;
    field_info     fields[fields_count];
    u2             methods_count;
    method_info    methods[methods_count];
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

魔数+副版本号+主版本号+常量池计数器+常量池数据区+访问标志(access flags)+类索引(this_class)+父类索引(super_clas)+接口计数器(interfaces_count)+fields+methiods+类文件属性(attributes)

- **[Magic Number](https://en.wikipedia.org/wiki/Magic_number_(programming))**: 0xCAFEBABE
- **Version of Class File Format**: the minor and major versions of the class file
- **Constant Pool**: Pool of constants for the class
- **Access Flags**: for example whether the class is abstract, static, etc.
- **This Class**: The name of the current class
- **[Super Class](https://en.wikipedia.org/wiki/Superclass_(computer_science))**: The name of the super class
- **[Interfaces](https://en.wikipedia.org/wiki/Interface_(object-oriented_programming))**: Any interfaces in the class
- **Fields**: Any fields in the class
- **[Methods](https://en.wikipedia.org/wiki/Method_(computing))**: Any methods in the class
- **Attributes**: Any attributes of the class (for example the name of the sourcefile, etc.)

#### 类的加载过程

1. Loading

   通过类的全限名获取此类的二进制字节流

   将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构

   在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口

2. Linking(Verification => Preparation => Resolution)

   Verification ：

   * 目的在于确保Class文件的字节流中包含信息符合当前虚拟机的要求，保证被加载类的正确性，不会危害虚拟机自身安全。
   * 主要包括四种验证，文件格式验证，元数据验证，字节码验证，符号引用验证

   Preparation ：

   * 为类变量分配内存并且设置该类变量的默认初始值，即零值。

   Resolution：

   * 讲常量池内的符号引用转换为直接应用的过程。
   * 事实上，解析操作往往会伴随着JVM在执行完初始化之后再执行。

3. Initialization