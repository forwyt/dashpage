+++
date = '2016-09-14T15:44:54+08:00'
draft = true
title = 'ios内存原理（3）'
tags = ["binary","code","ios"]
+++

![](/images/29e9bff778c21ba1fce2e366c566cc14.watch.webp)
虽然ARC时代不需要手动管理内存，但是我们仍然要注意循环引用和CoreFoundation对象的引用计数问题...
<!-- more -->


虽然ARC时代不需要手动管理内存，但是我们仍然要注意循环引用和CoreFoundation对象的引用计数问题
### MRC

|  对象操作 | OC中对应的方法  | 对应的引用计数的化 |
|  :-:  | :-:  | :-:  |
| 持有对象  | retain | +1 |
| 释放对象  | release | -1 |
| 废弃对象  | dealloc | 0 |

### ARC

根据指针来操作引用
在ARC模式下，只要没有强指针（强引用）指向对象，对象就会被释放。在ARC模式下，不允许使用retain、release、retainCount等方法。并且，如果使用dealloc方法时，不允许调用[super dealloc]方法。

ARC模式下的property变量修饰词为strong、weak，相当于MRC模式下的retain、assign。strong :代替retain，缺省关键词，代表强引用。weak：代替assign，声明了一个可以自动设置nil的弱引用，但是比assign多一个功能，指针指向的地址被释放之后，指针本身也会自动被释放。

实现方式是在编译时期自动在已有代码中插入合适的内存管理代码以及在 Runtime 做一些优化。详细的工作原理见 上一篇 **内存管理(2)**

ARC提供四种与内存管理有关的变量标识符，分别是：


### 关键字
- __strong
>__strong 是默认使用的标识符。只有还有一个强指针指向某个对象，这个对象就会一直存活。
- __weak
>__weak 声明这个引用不会保持被引用对象的存活，如果对象没有强引用了，弱引用会被置为 nil
- _ unsafe _unretained
>__unsafe_unretained 声明这个引用不会保持被引用对象的存活，如果对象没有强引用了，它不会被置为 nil。如果它引用的对象被回收掉了，该指针就变成了野指针。
- __autoReleasing
>__autoreleasing 用于标示使用引用传值的参数（id *），在函数返回时会被自动释放掉。

变量标识符的用法如下，在类型和变量名之间：

```objc
Number* __strong num = [[Number alloc] init];
```
属性标识符
```objc
@property (assign/retain/strong/weak/unsafe_unretained/copy) Number* num
```


assign表明 setter 仅仅是一个简单的赋值操作，通常用于基本的数值类型，例如CGFloat和NSInteger。

| key | 含义 |
| --- | --- |
|  strong  |表明属性定义一个拥有者关系。当给属性设定一个新值的时候，首先这个值进行 retain ，旧值进行 release ，然后进行赋值操作。
| weak | 表明属性定义了一个非拥有者关系。当给属性设定一个新值的时候，这个值不会进行 retain，旧值也不会进行 release， 而是进行类似 assign 的操作。不过当属性指向的对象被销毁时，该属性会被置为nil |
| unsafe_unretained | 语义和 assign 类似，不过是用于对象类型的，表示一个非拥有(unretained)的，同时也不会在对象被销毁时置为nil的(unsafe)关系 |
| copy |类似于 strong，不过在赋值时进行 copy 操作而不是 retain 操作。通常在需要保留某个不可变对象（NSString最常见），并且防止它被意外改变时使用 |
| unsafe_unretained | （1）兼容性，iOS4之前没有weak只能用unsafe_unretained（2）性能考虑，性能比weak要好 |


### ARC下仍然需要注意的内存管理问题
循环引用问题。
当两个对象互相持有对方的强引用时，并且这两个对象的引用计数都不是0的时候，便造成了引用循环，从而使用计数后内存无法得到释放。可以从以下几个方面入手：

(1)注意变量作用域，使用 autorelease 让编译器来处理引用
Autorelase Pool 提供了一种可以允许你向一个对象延迟发送release消息的机制。当你想放弃一个对象的所有权，同时又不希望这个对象立即被释放掉（例如在一个方法中返回一个对象时），Autorelease Pool 的作用就显现出来了。所谓的延迟发送release消息指的是，当我们把一个对象标记为autorelease时:

```objc
NSString* str = [[[NSString alloc] initWithString:@"hello"] autorelease];
```
这个对象的 retainCount 会+1，但是并不会发生 release。当这段语句所处的 autoreleasepool 进行 drain 操作时，所有标记了 autorelease 的对象的 retainCount 会被 -1。即 release 消息的发送被延迟到 pool 释放的时候了。

(2)在 ARC 环境下，苹果引入了 @autoreleasepool 语法，不再需要手动调用 autorelease 和 drain 等方法。系统在 runloop 中创建的 autoreleaspool 会在 runloop 一个 event 结束时进行释放操作。我们手动创建的 autoreleasepool 会在 block 执行完成之后进行 drain 操作。
main.m 中 Autorelease Pool 的解释
```objc
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```
这里的 pool 有什么作用，能不能去掉之类? 在这里我们分析一下。根据苹果官方文档， UIApplicationMain 函数是整个 app 的入口，用来创建 application 对象（单例）和 application delegate。

尽管这个函数有返回值，但是实际上却永远不会返回，当按下 Home 键时，app 只是被切换到了后台状态。同时参考苹果关于 Lifecycle 的官方文档，UIApplication 自己会创建一个 main run loop，我们大致可以得到下面的结论：

- main.m 中的 UIApplicationMain 永远不会返回，只有在系统 kill 掉整个 app 时，系统会把应用占用的内存全部释放出来。因为UIApplicationMain 永远不会返回，这里的 autorelease pool 也就永远不会进入到释放那个阶段
- 假设有些变量真的进入了 main.m 里面这个 pool（没有被更内层的 pool 捕获），那么这些变量实际上就是被泄露的。这个 autorelease pool 等于是把这种泄露情况给隐藏起来了。UIApplication 自己会创建 main run loop，在 Cocoa 的 runloop 中实际上也是自动包含 autorelease pool 的，因此 main.m 当中的 pool 可以认为是没有必要的。

因为我们看不到更底层的代码，加上苹果的文档中不建议修改 main.m ，所以我们也没有理由就直接把它删掉。

### Autorelease Pool 与函数返回值

如果一个函数的返回值是指向一个对象的指针，那么这个对象肯定不能在函数返回之前进行 release，这样调用者在调用这个函数时得到的就是野指针了，在函数返回之后也不能立刻就 release，因为我们不知道调用者是不是 retain 了这个对象，如果我们直接 release 了，可能导致后面在使用这个对象时它已经成为 nil 了。

为了解决这个纠结的问题， Objective-C 中对对象指针的返回值进行了区分，一种叫做 retained return value，另一种叫做 unretained return value。前者表示调用者拥有这个返回值，后者表示调用者不拥有这个返回值，按照“谁拥有谁释放”的原则，对于前者调用者是要负责释放的，对于后者就不需要了。

按照苹果的命名习惯，以 alloc, copy, init, mutableCopy 和 new 这些方法打头的方法，返回的都是 retained return value，例如 [[NSString alloc] initWithFormat:]，而其他的则是 unretained return value，例如 [NSString stringWithFormat:]。我们在编写代码时也应该遵守这个习惯。

（2）使用弱引用(weak)

​ 弱引用虽然持有对象，但是并不增加引用计数，这样就避免了循环引用的产生。在 iOS 开发中，弱引用通常在 delegate 模式中使用。

​ 弱引用的实现原理是这样，系统对于每一个有弱引用的对象，都维护一个表来记录它所有的弱引用的指针地址。这样，当一个对象的引用计数为 0 时，系统就通过这张表，找到所有的弱引用指针，继而把它们都置成 nil。

从这个原理中，我们可以看出，弱引用的使用是有额外的开销的。虽然这个开销很小，但是如果一个地方我们肯定它不需要弱引用的特性，就不应该盲目使用弱引用。

（3）当实例变量完成工作后，将其置为nil

​ 在合理的位置主动断开环中的一个引用，使得对象得以回收，类似于手动管理，要求对何时释放以及使用内存等非常熟悉。

底层 Core Foundation 对象，需要自己手工管理它们的引用计数时。
对于这些底层 Core Foundation对象的引用计数的修改，要相应的使用 CFRetain 和 CFRelease 方法。

```objc
// 创建一个 CTFontRef 对象
CTFontRef fontRef = CTFontCreateWithName((CFStringRef)@"ArialMT", fontSize, NULL);
// 引用计数加 1
CFRetain(fontRef);
// 引用计数减 1
CFRelease(fontRef);
对于 CFRetain 和 CFRelease 两个方法，我们可以直观地认为，这与 Objective-C 对象的 retain 和 release 方法等价。
```
所以对于底层 Core Foundation 对象，我们只需要延续以前手工管理引用计数的办法即可。
### other
除此之外，还有另外一个问题需要解决。在 ARC 下，我们有时需要将一个 Core Foundation 对象转换成一个 Objective-C 对象，这个时候我们需要告诉编译器，转换过程中的引用计数需要做如何的调整。这就引入了bridge相关的关键字，以下是这些关键字的说明：
- __bridge: 只做类型转换，不修改相关对象的引用计数，原来的 Core Foundation 对象在不用时，需要调用 CFRelease 方法。
- __bridge_retained：类型转换后，将相关对象的引用计数加 1，原来的 Core Foundation 对象在不用时，需要调用 CFRelease 方法。
- __bridge_transfer：类型转换后，将该对象的引用计数交给 ARC 管理，Core Foundation 对象在不用时，不再需要调用 CFRelease 方法。

根据具体情况合理使用上面的 3 种转换关键字，就可以解决 Core Foundation 对象与 Objective-C 对象相对转换的问题了。

### 总结

虽然ARC时代不需要手动管理内存，但是我们仍然要注意循环引用和CoreFoundation对象的引用计数问题。如果对内存管理没有把握可以使用Instrument中的 Leaks 工具来检测循环应用。


