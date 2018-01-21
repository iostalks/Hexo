---
title: Objective-C 属性的实现原理
date: 2017-07-31 22:39:14
tags:
---

本文主要是通过 Objc 的源码分析，更加深入的了解 NSObject 对象内部属性的存储结构，源码来自[objc4-680.tar.gz](https://opensource.apple.com/tarballs/objc4/)，为了便于源码的定位，我已将源码 Clone 到[这里](https://github.com/iostalks/Analyze)，文章中出现的代码可点击跳转到相应的位置。

带着问题出发：
* 属性和实例变量的关系
* 属性和实例变量是如何存储的
* 为什么 Category 可以使用属性却不能新增实例变量

#### 属性的结构
在 Objective-C 2.0 中，类的存储结构存放在[objc-runtime-new.h](https://github.com/iostalks/Analyze/blob/master/objc4-680-runtime/runtime/objc-runtime-new.h#L1064)

``` C++
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags

	// ....
}
```

Class 的主要数据信息存储在`class_data_bits_t`结构体中，该结构体是`class_rw_t *`结构体的指针加上一些自定义的信息。

[class_data_bits_t 关键代码](https://github.com/iostalks/Analyze/blob/master/objc4-680-runtime/runtime/objc-runtime-new.h#L845)：

```C++
struct class_data_bits_t {
	uintptr_t bits;

	class_rw_t* data() {
		return (class_rw_t *)(bits & FAST_DATA_MASK);
   }
}
```

该结构体仅包含一个指针字段，并为我们方便的提供了返回`class_rw_t`指针的方法。

`FAST_DATA_MASK = 0x00007ffffffffff8UL`，指出`class_rw_t *`只取bits 的[3, 47] 位。
> 在 x86_64 构架上，Mac OS 只使用了其中的 47 位来为对象分配地址。而且因为地址要按字节对齐，所以后面的三个掩码为 0.

为了提高 64 位内存的利用率，最后三位单独作为其它的用途：

[!image](http://oox2n98ab.bkt.clouddn.com/objc-method-class_data_bits_t.png)

``` C++
#define FAST_IS_SWIFT           (1UL<<0)
#define FAST_HAS_DEFAULT_RR     (1UL<<1)
#define FAST_REQUIRES_RAW_ISA   (1UL<<2)
```

* isSwift`FAST_IS_SWIFT`用于判断当前是否为 Swift 类；
* hasDefaultRR：`FAST_HAS_DEFAULT_RR`当前类或者父类是否含有 `retain/release/autorelease/retainCount/_tryRetain/_isDeallocating/retainWeakReference/allowsWeakReference`方法；
* requiresRawIsa: 当前类是否使用纯指针表示；

Objc 类中的属性、方法还有遵循的协议都保存在`class_rw_t`中：

``` C++
struct class_rw_t {
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;

    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;
}
```

 `properties`是`property_array_t`数组，存储的元素类型为`property_t`或者`property_list_t`。[查看源码](https://github.com/iostalks/Analyze/blob/master/objc4-680-runtime/runtime/objc-runtime-new.h#L797)

`property_t`是一个结构体，`name`代表属性的名称，`attributes`代表它的类型以及特征。
``` C
struct property_t {
    const char *name;
    const char *attributes;
};
```

`class_rw_t`中还有一个跟它类似名字的结构体`class_ro_t`，根据命名规则推断，rw为读写，ro为只读。主类里面定义属性会有对应的实例变量，实例变量就存在于这个结构体列表`ivars`中。

```
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
    uint32_t reserved;

    const uint8_t * ivarLayout;
    
    const char * name;
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;
};
```

`ivar_list_t`是自定义数组，存储`ivar_t`结构体。同时这里还存在一个`baseProperties`表示属性的字段，这跟 `class_rw_t`中的`properties`关系会在下一节中解释。

```C++
struct ivar_t {
    int32_t *offset; 
    const char *name;
    const char *type;
};
```

从`ivar_t`命名就能够看出来 offset, name, type 分别代表地址偏移量，名称和类型。

#### 属性的初始化

前一节了解了类的结构，以及属性和实例变量在类中的位置，这一节以 Person 类为例子，探究它们是如何被初始化的。

``` Objective-C
// Person.h
@interface Person : NSObject {
    NSString *_address;
}
@property (nonatomic, strong) NSString *name;
@end

// Person.m
@interface Person()
@property (nonatomic, assign) int privateAge;
@end

@implementation Person
@end
```

在该类的 .h 中声明一个 name 公开属性和 _address 实例变量， .m 中声明一个 privateAge 私有属性。

类的属性对应有实例变量，编译器默认在属性前面加下划线作为对应的实例变量（可通过 @synthesize 修改）。Person 类包含的实例变量有 _address, _name, _privateAge。

听说类的实例变量是在编译期就确定的，我们使用 clang 命令来验证一下：

``` C++
clang -rewrite-objc Person.m
```

重写之后在一个近十万行代码的 Person.cpp 文件中找到关键代码：

``` C++
static struct _class_ro_t _OBJC_CLASS_RO_$_Person __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	0, __OFFSETOFIVAR__(struct Person, _address), sizeof(struct Person_IMPL), 
	(unsigned int)0, 
	0, 
	"Person",
	(const struct _method_list_t *)&_OBJC_$_INSTANCE_METHODS_Person,
	0, 
	(const struct _ivar_list_t *)&_OBJC_$_INSTANCE_VARIABLES_Person,
	0, 
	(const struct _prop_list_t *)&_OBJC_$_PROP_LIST_Person,
};

static struct /*_ivar_list_t*/ {
	unsigned int entsize;  // sizeof(struct _prop_t)
	unsigned int count;
	struct _ivar_t ivar_list[3];
} _OBJC_$_INSTANCE_VARIABLES_Person __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_ivar_t),
	3,
	{{(unsigned long int *)&OBJC_IVAR_$_Person$_address, "_address", "@\"NSString\"", 3, 8},
	 {(unsigned long int *)&OBJC_IVAR_$_Person$_privateAge, "_privateAge", "i", 2, 4},
	 {(unsigned long int *)&OBJC_IVAR_$_Person$_name, "_name", "@\"NSString\"", 3, 8}}
};
```

实例变量被保存在`_OBJC_$_INSTANCE_VARIABLES_Person`结构体中，该结构体用于`_class_ro_t`的初始化。由此可以看出类的实例变量大小在编译器就被确定了。

在编译完成之后，程序会加载 Runtime 库，进入运行期。运行期会进一步对类的结构进行初始化。
 
在 [objc-runtime-new.mm](https://github.com/iostalks/Analyze/blob/master/objc4-680-runtime/runtime/objc-runtime-new.mm#L1714) 中有一个`static Class realizeClass(Class cls)`方法，实现的功能有：
* 调用`class_data_bits_t` 的 `data()`方法，将`class_rw_t`指针强转为`class_ro_t`；
* 初始化`class_rw_t`结构体；
* 设置结构体的`ro`和`flags`；
* 最后为类设置正确的 data；

``` C++
const class_ro_t *ro = (const class_ro_t *)cls->data();
class_rw_t *rw = (class_rw_t *)calloc(sizeof(class_rw_t), 1);
rw->ro = ro;
rw->flags = RW_REALIZED|RW_REALIZING;
cls->setData(rw);
```

这里之所以将`class_rw_t`强转为`class_ro_t`，是因为编译完成后类的`data()`返回值是`_class_ro_t`类型的，并且 `_class_ro_t`和`class_ro_t`的内存布局完全一致。

``` C++
struct _class_ro_t {
	unsigned int flags;
	unsigned int instanceStart;
	unsigned int instanceSize;
	unsigned int reserved;
	const unsigned char *ivarLayout;
	const char *name;
	const struct _method_list_t *baseMethods;
	const struct _objc_protocol_list *baseProtocols;
	const struct _ivar_list_t *ivars;
	const unsigned char *weakIvarLayout;
	const struct _prop_list_t *properties;
};

```

`realizeClass`方法对类进行第一次初始化，正确的设置了`rw`的`ro`字段。`rw`中的其他字段在`methodizeClass`方法中根据 `ro`进行二次初始化。有兴趣的同学可以自行查看[这里](https://github.com/iostalks/Analyze/blob/master/objc4-680-runtime/runtime/objc-runtime-new.mm#L677)。

至此，类完整的实现了初始化，在`class_rw_t`保存类的所有属性，`class_ro_t`保存类的基础属性和实例变量。`class_ro_t` 是只读的结构，所以在运行期无法动态的为类添加实例变量。

然而，我们可以在 Category 中为类添加属性，不能添加实例变量，是因为 Category 是在运行期决议的， Category 中声明的属性，会被动态的添加到可读写`class_rw_t`结构体的`properties`字段，而不会添加到只读`class_ro_t`结构体。


[重度参考](https://github.com/Draveness/Analyze/blob/master/contents/objc/%E6%B7%B1%E5%85%A5%E8%A7%A3%E6%9E%90%20ObjC%20%E4%B8%AD%E6%96%B9%E6%B3%95%E7%9A%84%E7%BB%93%E6%9E%84.md)

