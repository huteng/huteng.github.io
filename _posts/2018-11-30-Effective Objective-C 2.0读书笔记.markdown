---
layout: post
category: "read"
title:  "Effective Objective-C 2.0读书笔记"
tags: [iOS]
---

本月重温了《Effective Objective-C 2.0》，受益颇多。记录盲区与重点知识，以做备忘。

## 三、多用字面量语法，少用与之等价的方法

用字面量语法的好处：

1. 代码精简。

2. 异常检查。

   ```
   NSObject *obj1 = [[NSObject alloc] init];
   NSObject *obj2 = nil;
   NSObject *obj3 = [[NSObject alloc] init];
   
   // 若有nil，arrayA只有一个obj，arrayB会抛出异常。
   NSArray *arrayA = [NSArray arrayWithObjects:obj1, obj2, obj3, nil];
   NSArray *arrayB = @[obj1, obj2, obj3];
   ```


## 四、多用类型常量，少用 #define 预处理指令

#### 异同点：

1. 重复定义。宏允许重复定义，类型常量不会。
2. 类型检查。宏没有类型检查，类型常量有类型检查。
3. 宏功能强大。

#### static 关键字的作用：

1. 用在全局变量，表明这个变量在每个**编译单元**有独自的实例。
2. 用在函数里的局部变量，表明它的生存周期其实是全局变量，但仅在函数内可见。
3. 用在类成员，表明成员或者方法是类的，而不是对象实例的。

#### static const 与 const 的区别：

1. static const 修饰的关键字，是静态不可变全局变量，表明这个变量在每个**编译单元**有独自的实例，不能在外部被引用。
2. const 修饰的关键字，是不可变全局变量，表明在**全局符号表**有定义。其他文件可通过 extern 关键字引用该定义。

拓展：【全局符号表】



## 五、用枚举表示状态、选项、状态码

* C++11 后，可指定枚举类型，引入了前置声明功能。
* Objective-C 中，多用 NS_ENUM 与 NS_OPTIONS。
* NS_ENUM 与 NS_OPTIONS 的区别在于，后者省去了做 `|`时的强制类型转换。（C++中不允许类型隐式转换）



## 六、理解"属性"这一概念

* @property 会默认为类生成实例变量、getter方法、setter方法。

* 若 @perperty 声明了 @dynamic，则不生成这些信息，且编译能通过。objc 将在运行时去查找相关定义。

* C++ 和 Objective-C 调用实例变量的实现不同。

  * C++ 访问实例变量，在汇编层面访问对应内存的偏移值。（编译时确定）

  * Objective-C 访问实例变量， 将在类对象里查找成员的偏移量，在运行时动态决定访问的具体地址。（运行时确定）

    ```
    0x105b036ea <+58>:  movq   %rax, -0x10(%rbp)
    // rax存放的是实例对象的地址值。
    0x105b036ee <+62>:  movq   -0x10(%rbp), %rax
    // rip是类对象的地址，rsi存放的是偏移量
    // 这一步比较关键，动态的决定了偏移值。
    0x105b036f2 <+66>:  movq   0x26af(%rip), %rsi        ; WWKPerson.height
    // 赋值操作，将常量10赋值到内存地址
    0x105b036f9 <+73>:  movq   $0xa, (%rax,%rsi)
    ```


## 八、理解“对象等同性”这一概念

1. 异同点：
   1. `==`是比较指针地址，当指针的对象相同时，返回为YES。
   2. isEqual:`默认实现是判断`A == B。
   3. `hash`默认实现是返回对象地址值。
2. 若要实现自定义对象的比较，应该注意：
   1. 实现`isEqual:`方法，利用自定义条件比较一致性。
   2. 实现`hash`方法，相同对象返回同样的hash值。
   3. 注意`hash`的实现效率和哈希碰撞，多以成员的hash做`^`运算来实现。
   4. 实现`isEqual:`时，应先判断对象，再判断值。
3. `NSMutableSet`的容器中，判断对象相同的条件是：
   1. `isEqual:`为YES。
   2. `hash`相同。
4. `NSMutableDictionary`的 key 比较：
   1. 调用 NSCopying 的 copyWithZone: 方法获取对象。
   2. 调用 key 的`hash`决定放在哪个`bucket`中。（提升效率用）
   3. 利用 key 的 `isEqual:`判断 `key`使用相同。（注意不会依赖于 hash 的值）。



## 九、以“类族模式”隐藏实现细节

* NSArray、NSMutableArray实际上都是类族基类，其中并没有持有数据。



## 十、关联对象存放自定义数据

* `NSDcitionary`的 key 用 `isEqual:`来判断等同性。`objc_setAssociatedObject`的 key 利用`==`来判断等同性。



## 十一、理解 objc_msgSend 的作用

* 类对象有方法列表缓存。
* 消息发送有几种情况：
  * objc_msgSend 处理普通消息发送。
  * objc_msgsend_stret 处理返回值是结构体，且 CPU 的寄存器能够存放下返回类型的情况。若结构体太大，则交由另外的函数派发，返回值交由栈帧上的某个变量来处理。（实测这里好像有变动，不是书里说的这样了）
  * objc_msgSend_fpret 处理返回值是浮点数的情况。
  * objc_msgSendSuper 处理给超类发消息的情况。在其中也有两个与 `objc_msgsend_stret`和 `objc_msgSend_fpret`等效的方法。
* 尾调用优化(tail-call optimization)。



## 十二、理解消息转发机制

1. 动态方法解析（dynamic method resolution）。
   1. `+(BOOL) resolveInstanceMethod:(SEL)selector`或`+(BOOL)resolveClassMethod:(SEL)selector`
   2. 通常配合 `@dynamic` 和 `class_addMethod`使用。
2. 备援接受者（replacement receiver）。
   1. `-(id)forwardingTargetForSelector(SEL)selector`
   2. 可模拟多重继承。
   3. 这一步无法修改发送消息，若需修改，则需到下一步处理。
   4. 配合 NSProxy 使用。
3. 完整的消息转发机制（full forwarding mechanism）。
   1. `-(void)forwardInvocation:(NSInvocation *)invacation`
   2. 通常在该方法里，修改调用参数或更换 `SEL`。
   3. `- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector`中覆盖方法签名，该方法在第2步和第3步之间调用。



## 十三、用“方法调配技术”调试“黑盒方法”

* 又称为：Method Swizzling
  * IMP定义为：`id (*)IMP(id, SEL, ...)`
  * Method定义为：`typedef struct objc_method *Method`，是IMP的封装。
  * selector和IMP的关系，通过选择器表（selector table）来映射。修改关系表，实际上修改的就是 selector table。
* 多与 `+(void)load` 配合使用。



## 十四、理解“类对象”的用意

* `isMemberOfClass:`判断对象是否为某个特定类的实例。
*  `isKindOfClass:`判断对象是否为某类或其父类的实例。
* 比较类对象是否等同，尽量使用上述方法来判断，而不是直接使用`==`判断，因为前者可能实现了消息转发工功能（譬如NSProxy的子类）。



## 十七、实现 description 方法

* 覆盖 description 用于 log 时使用。
* 覆盖 debugDescription 用于 LLVM `po` 时使用。



## 十八、尽量使用不可变对象

* 对外暴露的对象，若不希望修改，则声明 `@property` 为 `readonly`。若内部希望修改，则在类扩展里声明为 `readwrite`。
* 即使声明为了 readonly，调用者仍可以通过 `KVC`，内存偏移硬编码等方式进行修改，但是通常不建议这样做。
* 如果内部是可变容器(NSMutableArray)，返回值为不可变容器(NSArray)，外层实际上拿到的是 NSArray 指针，但是指向了 NSMutableArray 的内容，存在被调用者修改的风险。作为类提供者，比较合理的是，返回时进行一次 copy 操作。



## 二十一、理解 Objective-C 错误模型

* `NSException`多用于非常严重的错误，当使用 NSException 时，程序会因为异常退出。
* `NSError`用来传递错误信息。
  * 通常传入参数为`NSError **`，相当于分配了堆上的指针，编译器默认加`autorelease`。
  * NSError通常包含三个信息：`Error Domain`，`Error Code`，`User Info`。



## 二十二、理解 NSCopying 协议

* NSCopying协议对应的`copy`方法，NSMutableCopy对应的`mutableCopy`方法。
* 默认情况下，copy返回不可变对象，mutableCopy返回可变对象。
* copy与mutableCopy默认都是浅拷贝。



## 二十三、通过委托和数据源协议进行对象间通信

* 用 `bitfield`缓存方法响应`respondToSelector:`结果，加快判断。

  ```objc
  struct delegateFlags {
      unsigned char didChangeNameWith : 1;
      unsigned char didUpgradeAgeWith : 1;
      unsigned char didAddNewFriendWith : 1;
  } _delegateFlags;
  
  - (void)setDelegate:(id<WWKPersonDelegate>)delegate {
      _delegate = delegate;
      _delegateFlags.didChangeNameWith = [self.delegate respondsToSelector:@selector(person:didChangeNameWith:)];
      _delegateFlags.didUpgradeAgeWith = [self.delegate respondsToSelector:@selector(person:didUpgradeAgeWith:)];
      _delegateFlags.didAddNewFriendWith = [self.delegate respondsToSelector:@selector(person:didAddNewFriendWith:)];
  }
  ```


## 二十九、理解引用计数

* 对象所占的内存在"deallocated"后，只是放回“可用内存池”(avaiable pool)。如果访问野指针时，内存没有被覆盖，那么该对象仍然有效，此时应用也不会崩溃。因此，防止野指针比较好的方式是，内存释放后，把对应的指针也置nil。



## 三十、以 ARC 简化引用计数

* ARC 的优化：
  * ARC 的引用计数实际上还是要执行的，只不过保留和释放操作是ARC在编译阶段自动添加的。通过汇编可以明确看到有调用 release 方法，调用版本是底层的 C 函数，而不是通过消息发送机制。
  * ARC 会通过优化调用方法`objc_autoreleaseReturnValue、objc_retainAuthreleaseReturnValue`等，来减少 retain，release的调用。
  * ARC 还有其他的优化策略。

* dealloc：

  * 析构时，`dealloc` 会调用编译器会自动生成的`.cxx_destruct`方法，在其中释放对象，包括 C++ 成员对象。

  * 手动在堆里分配的内存与`CoreFoundation`的对象，仍需要在 dealloc 中主动释放。

  * dealloc 调用的线程不一定，取决于引用计数置0时的线程。所以在dealloc里做只能主线程访问的操作时，需要额外小心。

  * 程序异常退出时，dealloc不一定会被调用。


## 三十四、以“自动释放池块”降低内存峰值

* 利用 `@autoreleasepool`降低循环里的内存峰值。



## 三十五、用“僵尸对象”调试内存管理问题

* 开启`NSZombieEnable`调试野指针问题。当内存被标记回收时，不会真正释放，而转化为特殊的`僵尸对象`。当向这种对象发送消息时，系统会抛出异常，方便调试。
* 僵尸对象原理是，当实例对象被释放时，将对象dealloc的实现替换成特殊实现。将该实例对象的isa指向新创建的僵尸类对象。僵尸类对象会接受所有的消息，打印log并且终止应用程序。



## 三十七、理解 "block" 这一概念

* block 实际是个对象。
* 注意 block 分配在哪里。（堆，栈，全局区）



## 第四十一、多用派发队列，少用同步锁

* @synchronized，底层实现为：

  ```
  // 忽略异常处理代码
  objc_sync_enter(self)
  id retVal = @"111";
  objc_sync_exit(self);
  return retVal;
  ```

  在`objc_sync_enter`内部，多个@synchronized会抢夺同一个锁。

* 处理多线程读写：
  * 方式一：加锁。简单场景使用。
  * 方式二：GCD串行队列，同步读，同步或异步写。
  * 方式三：GCD并发队列配合栅栏函数，同步读，barrier异步写。



## 第四十三、掌握 GCD 及操作队列的使用时机

* NSOperationQueue相比较于 GCD 队列的优势：
  * 取消指定任务。
  * 设置任务粒度的优先级。
  * 指定任务的依赖关系。
  * 通过 KVO 来观察任务的属性。
  * 可重用 NSOperation 对象。