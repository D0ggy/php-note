[toc]

# <span id="basic-structure">Basic structure

A zval (short for “Zend value”) represents an arbitrary PHP value. As such it is likely the most important structure in all of PHP and you’ll be working with it a lot. This section describes the basic concepts behind zvals and their use.

zval （“Zend 值”的缩写）表示任意 PHP 值。因此，它可能是所有 PHP 中最重要的结构，并且您将大量使用它。本节描述 zvals 背后的基本概念及其用法。

## <span id="types-and-values">Types and values

Among other things, every zval stores some value and the type this value has. This is necessary because PHP is a dynamically typed language and as such variable types are only known at run-time and not at compile-time. Furthermore the type can change during the life of a zval, so if the zval previously stored an integer it may contain a string at a later point in time.

除其他外，每个 zval 存储一些值以及该值具有的类型。这是必须的，因为 PHP 是一种动态类型化的语言，因此变量类型仅在运行时才知道，而在编译时才知道。此外，类型可以在 zval 的生存期内更改，因此，如果 zval 先前存储了整数，则它可能在以后的某个时间点包含字符串。

The type is stored as an integer tag (an unsigned int). It can be one of several values. Some values correspond to the eight types available in PHP, others are used for internal engine purpose only. These values are referred to using constants of the form `IS_TYPE`. E.g. `IS_NULL` corresponds to the null type and `IS_STRING` corresponds to the string type.

The actual value is stored in a union, which is defined as follows: