=============================
第二章: PHP变量在内核中的实现
=============================
.. highlight:: c++


所有编程语言都会提供数据的存取机制, PHP也不例外. 尽管许多语言都需要先定义所有变量并且变量的数据类型无法改变, PHP允许码畜们自由的(在运行期)创建变量和改变变量的数据类型, 还能根据需求自动转换数据类型.

因为看这个文档的肯定已经使用过PHP了, 所以弱类型什么的, 你肯定不陌生了. 而PHP是用C写成的, 本章将讲述PHP的数据是如何存放在强类型语言C中的.

当然, 存放数据只是前一半. 为了保持数据, 每块数据都需要一个标签和一个容器. 从用户的角度来看, 差不多就是变量名和作用域.


数据类型
========

PHP中最基本的数据存储单元是众所周知的\ **ZVAL**\ , 定义在\ ``Zend/zend.h``\ 中的结构体,  结构定义如下:\ ::

    typedef struct _zval_struct {
        zvalue_value value;
        zend_uint refcount;
        zend_uchar type;
        zend_uchar is_ref;
    } zval;

从直觉就能知道大部分成员的数据类型: **refcount**\ 是\ ``unsigned int``, **type**\ 和\ **is_ref**\ 是\ ``unsigned char``. 成员\ **value**\ 则是一个联合结构体, PHP5中是这样定义的\ ::

    typedef union _zvalue_value {
        long lval;
        double dval;
        struct {
            char *val;
            int len;
        } str;
        HashTable *ht;
        zend_object_value obj;
    } zvalue_value;


这种设计允许Zend引擎在一个单独的结构体中存储很多不同数据类型的PHP变量.

Zend目前(2006年)定义了8种数据类型, 如下表所示:

.. csv-table:: **表格2.1 Zend/PHP使用的数据类型**\ 
    :header: "类型", "存在目的"
    :widths: 15, 1000
    
    "IS_NULL", "这个类型是声明之后自动赋给未初始化的变量, 并且可以显式的指定为系统内置常量NULL. 这个值意味着非常独特的'没有值', 跟布尔值\ **FALSE**\ 和整型\ **0**\ 都不一样"
    "IS_BOOL", "布尔变量只能是\ **TRUE**\ 或者\ **FALSE**\ . 用户态的条件表达式结构(**if**, **while**, **ternary**, **for**)在执行的时候总是隐式的将判断结果转换为布尔类型"
    "IS_LONG", "PHP整型的数据类型用当前系统有符号长整形来存储. 在大多数32位平台上取值范围是 ``-2147483648`` 到 ``+2147483647``, 少数情况下, 当一个用户脚本尝试保存一个超出此范围的数值, 系统会自动转换为double类型(双精度类型 **IS_DOUBLE**)"
    "IS_DOUBLE", "浮点类型的数据类型使用当前系统的有符号双精度类型(signed double). 而众所周知的是计算机无法精准的表达浮点数, 取而代之的是用科学计数法来保存某个精度的浮点数. 1/3然后*3的结果是0.9999999..., 并不等于1.(这些限制存在于常见的32位系统, 系统和系统间有个体差异)"
    "IS_STRING", "PHP's most universal data type is the string which is stored in just the way an experienced C programmer would expect. A block of memory, sufficiently large to hold all the bytes/characters of the string, is allocated and a pointer to that string is stored in the host zval.
    What's worth noting about PHP strings is that the length of the string is always explicitly stated in the zval structure. This allows strings to contain NULL bytes without being truncated. This aspect of PHP strings will be referred to hereafter as binary safety because it makes them safe to contain any type of binary data.
     
      Note that the amount of memory allocated for a given PHP string is always, at minimum, its length plus one. This last byte is populated with a terminating NULL character so that functions that do not require binary safety can simply pass the string pointer through to their underlying method.
       "
    "IS_STRING", ""
