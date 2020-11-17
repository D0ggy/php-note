## Internal types

In this chapter we will detail the special types used internally by PHP. Some of those types are directly bound to userland PHP, like the “zval” data structure. Other structures/types, like the “zend_string” one, is not really visible from userland point of view, but is a detail to know if you plan to program PHP from inside.

在本章中，我们将详细介绍 PHP 内部使用的特殊类型。 其中一些类型直接绑定到用户级 PHP，例如 “zval” 数据结构。从用户区的角度来看，其他结构/类型（例如 “zend_string” 结构）并不是真正可见的，但是它是一个详细信息，用于了解您是否打算从内部对 PHP 进行编程。

Contents:

* [Zvals](_1_Zvals/README.md)
    * [Basic structure](_1_Zvals/README.md#basic-structure)
    * [Zval memory management and garbage collection](_1_Zvals/README.md#zval-memory-management-and-garbage-collection)
    * [Casts and operations on zvals](_1_Zvals/README.md#casts-and-operations-on-zvals)
    
Strings management
Strings management: zend_string
smart_str API
PHP’s custom printf functions
The Resource type: zend_resource
What is the “Resource” type?
Resource types and resource destruction
Playing with resources
Reference counting resources
Persistent resources
HashTables: zend_array
Functions: zend_function
Objects and classes