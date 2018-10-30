---
layout: post
category: "read"
title:  "Objective-C的类和对象模型"
tags: [iOS]
---

本文从 C++ 的角度出发，讲解和剖析 Objective-C 的类和对象模型。

### 一、基础知识储备

Objective-C 的底层是用 C/C++ 实现的，在系统讲解 Objective-C 的底层结构之前，我们先来温故一些基础知识。

#### 1、堆空间、栈空间

程序的内存一般分为五块，`堆空间`，`栈空间`，`代码区`，`字符常量区`，`全局区`。这里我们拎出开发中经常接触到的堆空间和栈空间来讲解。

栈空间的内存，系统自动管理，出作用域后回收内存。

堆空间的内存，需要手动管理，多与`new`，`delete`关键字搭配使用。

```
class Foo {
public:
    int value1;
    int value2;    
    void function() {}
};
void test () {
    // 1. 栈空间，函数作用域外自动回收内存
    Foo foo;
    
    // 2. 堆空间，需手动管理内存
    // 需注意指针本身的内存在栈空间，指针指向的内容在堆空间
    Foo *pFoo = new Foo();
    delete pFoo;
}
```

在实际的开发过程中，需注意内存分配在栈空间或堆空间。而在Objective-C中，使用`new`、`alloc`分配的对象都在堆空间上。

#### 2、函数地址

在讲解函数地址前，先抛出一个疑问，上面代码的`class Foo`，其中有两个`int`类型的成员value1、value2，一个成员方法 function()。那么用该类实例化的对象，内存占用多大呢？

答案：我们用`sizeof(foo)`可以得到，对象 `foo`占用了8个字节的内存空间，刚好是两个`int`类型相加。

那么这里产生了另外一个疑问，成员方法不占用内存大小吗？--- 答案是否定的。

完整的解释是，成员方法全局只存在一份，拥有全局唯一的函数地址；成员方法会占用全局内存大小，但是不占用实例化对象的内存空间。这个小知识点比较重要，在后续讲解 Objective-C 底层结构时也会提到。

#### 3、struct 和 class 的区别

`struct` 和 `class` 都能用来声明类，他们之间唯一的区别是对成员的权限管理不同，`struct`默认权限是`public`，`class`的默认权限是`private`。而Objective-C的类底层结构都会声明为 struct。

### 二、Objective-C 对象模型

Objective-C 的底层是用 C/C++ 实现的，在C++之上，Objective-C 封装了一层系统实现，实现了动态消息转发机制。不同于C++，对于 Objective-C 的对象，可以分为三种：

* instance对象，也称为实例对象。
* class对象，也称为类对象。
* meta-class对象，也称为元类对象。

#### 1、实例对象

每一个对类发送 `alloc` 消息，都会产生实例对象。

```objective-c
@interface HTObject : NSObject {
    int _value;
}
- (void)function;
@end
@implementation HTObject
- (void)function {}
@end

void test() {
	// 产生实例对象
	HTObject *object = [[HTObject alloc] init];
}
```

假设输入文件为main.mm，输出文件为main.cpp，利用命令`xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.mm -o main.cpp`，将Objective-C代码转换成C++代码，关键代码转化为：

```c++
typedef struct objc_class *Class;
struct NSObject_IMPL {
	Class isa;
};
struct HTObject_IMPL {
	struct NSObject_IMPL NSObject_IVARS;
	int _value;
};
// @implementation HTObject
static void _I_HTObject_function(HTObject * self, SEL _cmd) {

}
// @end
```

从转化后C++代码中，我们可以得到三个很关键的信息：

1. Objective-C 的类底层是用`struct`来实现的。
2. Objective-C的实例对象中，在内存中存储了 `isa` 指针和成员变量。
3. 成员方法不占用实例对象的内存。

好，讲到这里得知了，实例对象仅存储了成员变量的值，那么成员方法和属性信息存在放哪里呢？在讲解这个知识点之前，我们先总览 Objective-C 的对象关系图。

#### 2、对象关系图

网上有张[介绍图](http://www.sealiesoftware.com/blog/class%20diagram.pdf)非常系统的表示实例对象、类对象、元类对象三者的关系。

![](http://ronhu.me/img/class_relationship.png)

剖析下这张图需要注意什么：

1. Objective-C 存在三种对象，实例对象（instance），类对象（class），元类对象（meta）。
2. 三种对象的内部都有`isa`指针，实例对象的`isa`指针指向类对象，类对象的`isa`指针指向元类对象，元类对象的`isa`指针指向`Root Class`的元类对象，即`NSObject`的元类对象。（Objective2.0之后，isa 间接存储指向地址）
3. `NSObject` 的**元类对象**的`superclass`指针指向了 `NSObject`的**类对象**。
4. 对象方法的查找，先从**实例对象**的`isa`指针找到**类对象**，若**类对象**不存在该方法，则通过superclass找父类。
5. 类方法的查找，先从**类对象**的`isa`指针找到**元类对象**，若**元类对象**不存在该方法，则通过superclass找父类。

好，看到这里，就可以推导出结论：

1. 实例对象仅存储了成员变量和指向类对象的 `isa` 指针。
2. 成员方法的函数地址等存储在类对象和元类对象里。

下面我们来看看类对象和元类对象的底层结构是怎样的。

#### 3、类对象

Objective-C 每个类定义，在内存中都存在类对象。

```objective-c
HTObject *object = [[HTObject alloc] init];
// 类对象
Class objectClass1 = [object class];
Class objectClass2 = object_getClass(object);
```

类对象的类型为`Class`，通过`typedef struct objc_class *Class`可以得知，`Class` 是指向 `struct objc_class` 的指针。 `objc_class`的定义可通过苹果开源的[源码](https://opensource.apple.com/tarballs/objc4/)来窥探一下:

```objective-c
struct objc_object {
private:
    // isa指针
    isa_t isa;
public:
	// ... 其他辅助方法
}

struct objc_class : objc_object {
	// 父类指针
    Class superclass;
    // 方法缓存
    cache_t cache;             
	// bits & FAST_DATA_MASK 可得到 class_rw_t *
    class_data_bits_t bits;    
    class_rw_t *data() { 
        return bits.data();
    }
	// ...其他辅助方法
}

// 类信息
struct class_rw_t {
    uint32_t flags;
    uint32_t version;
    // 类只读信息，包含类名，成员变量列表等不可变信息。
    const class_ro_t *ro; 
	
    // 方法列表
    method_array_t methods;
    // 属性列表
    property_array_t properties;
    // 协议列表
    protocol_array_t protocols;
 
    Class firstSubclass;
    Class nextSiblingClass;
    char *demangledName;    
	// ...其他辅助方法
}
```

通过源码，我们可以得知，`objc_class`里包含了几个关键信息：

1. isa指针，superclass指针。
2. 方法列表，属性列表，协议列表。
3. 类名，成员变量等不可变信息。
4. 其他辅助信息。

#### 4、元类对象

Objective-C 每个类定义，除了类对象外，在内存中也存在元类对象。

```objective-c
HTObject *object = [[HTObject alloc] init];
// 元类对象
Class objectMetaClass = object_getClass([object class]);
class_isMetaClass(objectMetaClass); 
```

元类对象的类型也为`Class`，底层结构和类对象一致，存储结构为`objc_class`。虽然存储结构一致，但是元类对象和类对象的用途却不同：

1. `methods`，元类存储的`类方法`，类对象存储的是`对象方法`。
2. `properties`，元类存储的`类属性`，类方法存储的`对象属性`。

所以，元类对象中主要是存储和类方法相关的信息，而类对象存储的是对象方法相关的信息。

### 三、总结与回顾

本文总结起来介绍了几个重要知识点：

1. Objective-C 的对象，分为三种：实例对象，类对象，元类对象。
2. Objective-C 的对象之间的关系通过 `isa` 来关联。
3. 类对象和元类对象底层结构一致，类对象主要存储成员方法相关的信息，元类对象主要存储类对象相关信息。

Objective-C的底层对象模型是基石，在掌握对象模型之后，我们能更好的领悟：

1. Objective-C 的消息发送机制。
2. Method Swizzing，方法动态替换。
3. KVC，KVO，Category 实现原理等。