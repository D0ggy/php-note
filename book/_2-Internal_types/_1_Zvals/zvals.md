[toc]

# <span id="basic-structure">Basic structure

A zval (short for “Zend value”) represents an arbitrary PHP value. As such it is likely the most important structure in all of PHP and you’ll be working with it a lot. This section describes the basic concepts behind zvals and their use.

zval （“Zend 值”的缩写）表示任意的一个 PHP 值。因此，它可能是所有 PHP 中最重要的结构，并且您将大量使用它。本节描述 zvals 背后的基本概念及其用法。

## <span id="types-and-values">Types and values

Among other things, every zval stores some value and the type this value has. This is necessary because PHP is a dynamically typed language and as such variable types are only known at run-time and not at compile-time. Furthermore the type can change during the life of a zval, so if the zval previously stored an integer it may contain a string at a later point in time.

除其他外，每个 zval 存储一些值以及值的类型。这是必须的，因为 PHP 是一种动态类型化的语言，因此变量类型仅在运行时才知道，在编译时不知道。此外，类型可以在 zval 的生存期内更改，因此，如果 zval 先前存储了整数，则它可能在以后的某个时间点包含字符串。

The type is stored as an integer tag (an unsigned int). It can be one of several values. Some values correspond to the eight types available in PHP, others are used for internal engine purpose only. These values are referred to using constants of the form `IS_TYPE`. E.g. `IS_NULL` corresponds to the null type and `IS_STRING` corresponds to the string type.
 
类型的存储是个整数标记（无符号 int）。它可以是几个值中的一个。一些值对应 PHP 中可用的八种类型，其他值仅用于内部引擎。使用 `IS_TYPE` 形式的常量引用这些值。例如，`IS_NULL` 对应空类型，`IS_STRING` 对应字符串类型。

The actual value is stored in a union, which is defined as follows:
```c
typedef union _zend_value {
    zend_long         lval;
    double            dval;
    zend_refcounted  *counted;
    zend_string      *str;
    zend_array       *arr;
    zend_object      *obj;
    zend_resource    *res;
    zend_reference   *ref;
    zend_ast_ref     *ast;
    zval             *zv;
    void             *ptr;
    zend_class_entry *ce;
    zend_function    *func;
    struct {
        uint32_t w1;
        uint32_t w2;
    } ww;
} zend_value;
```
To those not familiar with the concept of unions: A union defines multiple members of different types, but only one of them can ever be used at a time. E.g. if the `value.lval` member was set, then you also need to look up the value using `value.lval` and not one of the other members (doing so would violate “strict aliasing” guarantees and lead to undefined behaviour). The reason is that unions store all their members at the same memory location and just interpret the value located there differently depending on which member you access. The size of the union is the size of its largest member.

对于不熟悉共用体概念的人：共用体定义了多个不同类型的成员，但是一次只能使用其中一个。例如，如果设置了 `value.lval` 成员，则还需要使用 `value.lval` 而不是其他成员中的一个来查找值（这样做将违反“严格别名”保证，并导致未定义的行为）。原因是，共用体将所有成员存储在相同的内存位置，只是根据您访问的成员来不同地解释该位置的值。共用体的大小就是其最大成员的大小。

When working with zvals the type tag is used to find out which of the union’s member is currently in use. Before having a look at the APIs used to do so, let’s walk through the different types PHP supports and how they are stored:

zvals 在使用时，type 标签用于确定当前正在使用共用体的哪个成员。在查看用于实现此目的的 API 之前，让我们先了解一下 PHP 支持的不同类型以及它们的存储方式：

The simplest type is `IS_NULL`: It doesn’t need to actually store any value, because there is just one `null` value.

最简单的类型是 `IS_NULL`：它实际上不需要存储任何值，因为只有一个 `null` 值。

For storing numbers PHP provides the types `IS_LONG` and `IS_DOUBLE`, which make use of the `zend_long lval` and `double dval` members respectively. The former is used to store integers, whereas the latter stores floating point numbers.

为了存储这些成员，PHP 提供了 `IS_LONG` 和 `IS_DOUBLE` 类型，它们分别使用 `zend_long lval` 和 `double dval` 成员。前者用于存储整数，而后者用于存储浮点数。

There are some things that one should be aware of about the `zend_long` type: Firstly, this is a signed integer type, i.e. it can store both positive and negative integers, but is commonly not well suited for doing bitwise operations. Secondly, `zend_long` represents an abstraction of the platform long, so whatever the platform you’re using, `zend_long` weights 4 bytes on 32bit platforms and 8 bytes on 64bit ones.

关于 `zend_long` 类型，应该注意一些事情：首先，这是一个有符号整数类型，即它可以存储正整数和负整数，但通常不适合进行按位运算。其次，`zend_long` 代表平台 long 的抽象，因此无论您使用的平台是什么，`zend_long` 在 32 位平台上的权重为 4 字节，在 64 位平台上的权重为 8 字节。

In addition to that, you may use macros related to longs, `SIZEOF_ZEND_LONG` or `ZEND_LONG_MAX` f.e. See [Zend/zend_long.h](https://github.com/php/php-src/blob/c3b910370c5c92007c3e3579024490345cb7f9a7/Zend/zend_long.h) in source code for more information.

除此之外，您还可以使用与 long 相关的宏，`SIZEOF_ZEND_LONG` 或`ZEND_LONG_MAX` 等。有关更多信息，请参见源代码中的 [Zend/zend_long.h](https://github.com/php/php-src/blob/c3b910370c5c92007c3e3579024490345cb7f9a7/Zend/zend_long.h)。

The `double` type used to store floating point numbers is (typically) an 8-byte value following the IEEE-754 specification. The details of this format won’t be discussed here, but you should at least be aware of the fact that this type has limited precision and commonly doesn’t store the exact value you want.

用于存储浮点数的 `double` 类型（通常）是遵循 IEEE-754 规范的 8 字节值。此格式的详细信息不在这里讨论，但您至少应该意识到以下事实：该类型的精度有限，通常不会存储您想要的确切值。

Booleans use either the `IS_TRUE` or `IS_FALSE` flag and don’t need to store any more info. There exists what’s called a “fake type” flagged as `_IS_BOOL`, but you shouldn’t make use of it as a zval type, this is incorrect. This fake type is used in some rare uncommon internal situations (like type hints f.e).

布尔值使用 `IS_TRUE` 或 `IS_FALSE` 标志，不需要存储更多信息。存在标记为 `_IS_BOOL` 的所谓的“假类型”，但您不应将其用作 zval 类型，这是不正确的。这种伪类型用于一些罕见的内部情况（例如类型提示等）。

The remaining four types will only be mentioned here quickly and discussed in greater detail in their own chapters:

其余四种类型将仅在此处快速提及，并在其特定的章节中进行更详细的讨论：

Strings (`IS_STRING`) are stored in a `zend_string` structure, i.e. they consist of a `char *` string and an `size_t` length. You will find more information about the `zend_string` structure and its dedicated API into the ==string== chapter.

字符串（`IS_STRING`）存储在 `zend_string` 结构中，即它们由 `char *` 字符串和 `size_t` 长度组成。您可以在==string==一章中找到有关 `zend_string` 结构及其专用 API 的更多信息。

Arrays use the `IS_ARRAY` type tag and are stored in the `zend_array *arr` member. How the `HashTable` structure works will be discussed in the ==Hashtables== chapter.

数组使用 `IS_ARRAY` 类型标签，并存储在 `zend_array * arr` 成员中。  `HashTable` 结构体的工作方式将在 ==Hashtables== 一章中讨论。

Objects (`IS_OBJECT`) use the `zend_object *obj` member. PHP’s class and object system will be described in the ==objects== chapter.

对象（`IS_OBJECT`）使用 `zend_object *obj` 成员。PHP 的类和对象系统将在 ==objects== 一章中进行介绍。

Resources (`IS_RESOURCE`) are a special type using the `zend_resource *res` member. Resources are covered in the ==Resources== chapter.

资源（`IS_RESOURCE`）是使用 `zend_resource * res` 成员的特殊类型。资源在 ==Resources== 一章中介绍。

To summarize, here’s a table with all the available type tags and the corresponding storage location for their values:

总结一下，这是一张表格，其中包含所有可用的类型标签以及其值的对应存储位置：
|Type tag|Storage location|
|:---|:---|
|`IS_NULL`|none|
|`IS_TRUE` or `IS_FALSE`|none|
|`IS_LONG`|`zend_long lval`|
|`IS_DOUBLE`|`double dval`|
|`IS_STRING`|`zend_string *str`|
|`IS_ARRAY`|`zend_array *arr`|
|`IS_OBJECT`|`zend_object *obj`|
|`IS_RESOURCE`|`zend_resource *res`|

### Special type

You may see other types carried into the zvals, which we did not review yet. Those types are special types that do not exist as-is in the PHP language userland, but are used into the engine for internal use-case only. The zval structure has been thought to be very flexible, and is used internally to carry virtually any type of data of interest, and not only the PHP specific types we just reviewed above.

您可能会看到zvals中包含了其他类型，我们尚未进行审查。这些类型是特殊类型，在PHP语言用户领域中并不存在，而是仅用于内部用例的引擎中。 zval结构被认为是非常灵活的，并且在内部用于承载几乎任何感兴趣的数据类型，而不仅仅是我们上面刚刚回顾的特定于PHP的类型。

The special IS_UNDEF type has a special meaning. That means “This zval contains no data of interest, do not access any data field from it”. This is used for memory management purposes. If you see an IS_UNDEF zval, that means that it is of no special type and contains no valid information.

The zend_refcounted *counted field is very tricky to understand. Basically, that field serve as a header for any other reference-countable type. This part is detailed into the Zval memory management and garbage collection chapter.

The zend_reference *ref is used to represent a PHP reference. The IS_REFERENCE type flag is then used. Here as well, we dedicated a chapter to such a concept, have a look at the Zval memory management and garbage collection chapter.

The zend_ast_ref *ast is used when you manipulate the AST from the compiler. The PHP compilation is detailed into the Zend Compiler chapter.

The zval *zv is used internally only. You should not have to manipulate it. This works together with the IS_INDIRECT, and that allows one to embed a zval * into a zval. Very specific dark usage of such a field is used f.e to represent $GLOBALS[] PHP superglobal.

Something very useful is the void *ptr field. Same here : no PHP userland usage but internal only. You will basically use this field when you want to store “something” into a zval. Yep, that’s a void *, which in C represents “a pointer to some memory area of any size, containing (hopefully) anything”. The IS_PTR flag type is then used in the zval.

When you’ll read the objects chapter, you’ll learn about zend_class_entry type. The zval zend_class_entry *ce field is used to carry a reference to a PHP class into a zval. Here again, there is no direct usage of such a situation into the PHP language itself (userland), but internally you’ll need that.

Finally, the zend_function *func field is used to embed a PHP function into a zval. The functions chapter details PHP functions.