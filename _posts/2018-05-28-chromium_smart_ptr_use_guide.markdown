---
layout: post
category: "read"
title:  "Chromium智能指针使用指南"
tags: [C++]
---
#### 什么是智能指针？

智能指针是一种特殊类型的“局部对象”，表现如同裸指针，但是具备`离开作用域(out of scope)时主动释放所指向对象`的能力。因为C++没有垃圾回收机制，因此智能指针的特性显得非常重要。

下面是最常用智能指针类型`std::unique_ptr<>`的例子：
```C++
// 我们可以在构造std::unique_prt<>的时候传入指针
std::unique_ptr value(base::JSONReader::Read(data));
std::unique_ptr foo_ptr(new Foo(...));

// ...或者使用reset()
std::unique_ptr bar_ptr;      // 与 "Bar* bar_ptr = nullptr;" 相似.
bar_ptr.reset(new Bar(...));  // 此时 |bar_ptr| 不为空且持有对象 

// 我们可以用 () 检查std::unique_ptr<>是否为空
if (!value) return false;

// get() 访问持有的裸指针
Foo* raw_ptr = foo_ptr.get();

// 我们可以像使用裸指针一样调用std::unique_ptr<>的方法
DictionaryValue* dict;
if (!value->GetAsDictionary(&dict)) return false;
```

#### 为什么我们要使用智能指针？

即使对象的创建和析构时机不确定，使用智能指针确保我们能正确释放对象。无论方法里有再多且逻辑复杂的路径，智能指针总能确保局部变量正确的释放，且能明确对象的所有权，避免程序内存泄漏或者对象重复释放。最后，在方法调用时，需要明确指出对象拥有权的转移和结果。

#### 存在哪些类型的智能指针？

在Chromium里最常用的两种智能指针类型是`std::unique_ptr<>`和`scoped_refptr<>`。前者适用于单一所有权的对象，后者适用于引用计数的对象（然而，通常应该避免使用引用计数的对象）。如果你比较熟悉C++11，会发现`scoepd_refptr<>`和`std::shared_ptr<>`用法很相似。

[base/memory/](https://chromium.googlesource.com/chromium/src/+/master/base/memory/) 还定义了其余几种类型的对象：

- `linked_ptr<>` 用于在C++11之前存放智能指针对象，已被废弃。现在Chromium已经支持C++11了，我们不应该再使用`linked_ptr<>`了，而应该在STL容器里使用`std::unique_ptr<>`。
- `ScopedVector<>`也被废弃了。它是一种vector，并且持有容器内对象的所有权。请使用`std::vector<std::unique_ptr<>>`代替。
- `WeakPtr<>`实际上不是智能指针。它的表现像指针类型，但是并不能用来自动释放对象，通常用作追踪其它地方拥有的对象是否依然存活，当追踪对象释放时，`WeakPtr<>`会自动的置为`null`。（但是依然需要在解引用前判断是否为`null`，因为解引用`null WeakPtr<>`等于于解引用`null`，而不是no-op。）`WeakPtr<>`与C++11的`std::weak_ptr<>`作用比较相似，但是使用了不同的API并且少了许多使用限制。

#### 如何选择使用哪种智能指针？

- **单一所有权的对象**。使用`std::unique_ptr`。需要注意的是，`std::unique_ptr`持有的需要必须是非引用计数的，并且分配在堆上的对象。
- **无所有权的对象**。使用裸指针或者`WeakPtr<>`。注意`WeakPtr<>`只能在创建它的线程解引用（通常使用`WeakPtrFactory<>`）。如果你需要在对象释放前后立刻执行某些操作，那么可能使用callback或notification更适合，而不是`WeakPtr<>`。
- **引用计数的对象**。使用`scoped_refptr<>`，但是最好是重新考虑使用引用计数对象是否合理。引用计数对象很难明确拥有权和析构顺序，特别是在多线程环境中。总是有方法来重新设计引对象层级来避免引用计数的。限制每个类都只能在单个线程工作，并且使用`PostTask()`确保调用在正确的线程，这样有助于在多线程中避免引用计数。`base::Bind()`，`WeakPtr<>`等工具具备在对象释放时自动取消方法调用的能力。Chromium中依然有许多代码在使用引用计数对象，如果你看见Chromium中有代码这样做但并不代表这是合理的解决方案。
- **平台特定类型**。使用平台特定的对象，譬如`base::win::ScopedHandle`，`base::win::ScopedComPtr`，或者`base::mac::ScopedCFTypeRef`。需要注意的是这些类型使用方式可能和`std::unique_ptr<>`不同。

#### 不同类型指针间调用规定是怎样的？

[calling conventions section of the Chromium style guide](https://www.chromium.org/developers/coding-style?pli=1#TOC-Object-ownership-and-calling-conventions)有规定。下面列出一些常用的规定。

- 如果方法参数里使用`std::unique_ptr<>`	，说明该方法需占用传入参数的所有权，调用方需要使用`std::move()`来表明转移对象的所有权。需要注意的是，临时对象不需要调用`std::move()`转移所有权。

```C++
// Foo() 拥有 |bar| 的所有权.
void Foo(std::unique_ptr<Bar> bar);

...
std::unique_ptr<Bar> bar_ptr(new Bar());
Foo(std::move(bar_ptr));          // 调用后，|bar_ptr| 被置为 null.
Foo(std::unique_ptr<Bar>(new Bar()));  // 临时对象不需要调用std::move()
```

- 如果方法的返回值使用`std::unique_ptr<>`，说明调用方需要持有返回对象的所有权。这种情况下，当且仅当返回对象类型和临时对象的类型不同时，需要使用`std::move()`。

```C++
class Base { ... };
class Derived : public Base { ... };

// Foo 拥有|base|的所有权, 调用方拥有 返回值对象 的所有权
std::unique_ptr<Base> Foo(std::unique_ptr<Base> base) {
	if (cond) {
    	// 转移 |base| 的所有权给调用方
    	return base;                           
  	}
  
  // 注意这种场景下，方法运行结束时，|base|会被释放掉
  if (cond2) {
    // 临时对象不需要调用std::move()
    return std::unique_ptr<Base>(new Base()));  
  }
  std::unique_ptr<Derived> derived(new Derived());
  // 注意需要使用std::move()，因为|derived|的类型和返回值的类型不同。
  return std::move(derived);
}
```	

- 如果方法传入或者返回裸指针，表示无需所有权转移。Chromium在`std::unique_ptr<>`存在之前写的一些代码，或者不熟悉所有权转移的程序员写的代码，可能会在传入或者返回裸指针的时候也使用`std::move()`转移了所有权。但是这样做是不安全的，编译器并不能执行正确的表现。去掉这样的代码吧，方法传入或者返回裸指针时，绝对不要转移所有权。

#### 可以通过引用传递参数或者返回值吗？

不要这样做。

原理上来说，传入`const std::unique_ptr<T> &`参数并且不转移所有权比传入`T*`有优势，这样做可以防止调用方传入错误的参数(譬如把 int 转成了 T*)，而且调用方必须确保方法调用周期内传入对象不会被释放。但是，这样调用方就必须把传入对象生成在堆上，即使调用方原本可以使对象生成在栈上。这里传入裸指针相比传入`const std::unique_ptr<T> &`的好处是，可以将对象所有权的问题和对象生成的问题解耦。为了简洁和统一，我们避免开发人员去权衡这些利弊，总是使用裸指针就好了。

有个例外，在lambda表达式中，若将智能指针放在STL容器里作为参数传递，这里为了编译通过，必须使用`const std::unique_ptr<T> &`。

#### 我想使用STL容器用来持有指针对象。此时可以用智能指针吗？

可以。在C++11里，你可以将智能指针放入STL容器内。而且，不要再使用`ScopedVector<T>`了，使用`std::vector<std::unique_ptr<T>>`来替代。同样的，再也不要再使用`linked_ptr<T>`了，直接把智能智能放在STL容器里使用即可。

#### 引用资料

- [StackOverflow guide to smart pointers](http://stackoverflow.com/questions/106508/what-is-a-smart-pointer-and-when-should-i-use-one)
- [Wikipedia article on smart pointers](http://en.wikipedia.org/wiki/Smart_pointer)
- [Guide to smart pointer usage in WebKit](http://www.webkit.org/coding/RefPtr.html)
- [Smart Pointer Guidelines](https://www.chromium.org/developers/smart-pointer-guidelines)
