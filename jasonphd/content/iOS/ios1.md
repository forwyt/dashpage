+++
date = '2016-09-10T09:43:12+08:00'
draft = false
title = 'ios内存原理（1）'
tags = ["binary","code","ios"]
+++



![board1.jpg](/images/eb99c63605e608e7e685dc8c9d04603d.board1.webp)
在iOS的开发中我们接触到了ARC/MRC这些机制,为了更好的了解其中的原理,我们可以从官方的文档作为入口来看看这些名词背后更深入的意义.

<!-- more -->

在iOS的开发中我们接触到了ARC/MRC这些机制,为了更好的了解其中的原理,我们可以从官方的文档作为入口来看看这些名词背后更深入的意义.

## 01内存管理
![](https://jasonphd.com/pic/shentu.jpg)
基本内存管理规则内存管理模型基于对象所有权。任何对象都可以具有一个或多个所有者。只要一个对象至少具有一个所有者，它就会继续存在。如果对象没有所有者，则运行时系统会自动销毁它.

苹果给出了四条准则

- 管理任何由你自己创建的对象
- 通过retain来持有一个对象 (在对象方法/init方法中调用 )一般在下面两种情况
    - 当你想将一个对象存储为属性值时
   * 当你想防止一个对象被释放时
- 当你不再需要这个对象时候 请释放

```objc
    Person *aPerson = [[Person alloc] init];
    // ...
    NSString *name = aPerson.fullName;
    // ...
    [aPerson release];
```

tips
1.在iOS中涉及到对象持有的是基于一套 *NSObject protocol*协议 以及一些标准的方法名来实现的,例如alloc,newObject,copy,mutableCopy来实现的

2.管理对象,我们从Apple给出的两个例子

当你担心你的对象alloc后,在return之前调用[object release]导致对象被释放 那么采用autorelease,该方法会在返回对象之后调用release方法

```objc
- (NSString *)fullName {
    NSString *string = [[[NSString alloc] initWithFormat:@"%@ %@",
                                          self.firstName, self.lastName] autorelease];
    return string;
}
```
下面这种方法是在调用者调用此方法之后 对象被销毁之前达到将对象传递给外面的效果
```objc
- (NSString *)fullName {
    NSString *string = [NSString stringWithFormat:@"%@ %@",
                                 self.firstName, self.lastName];
    return string;
}
```
我们也可以通过ClassName 或者 id 去完成我们的逻辑操作例如&object指向内存地址 ,所以我们不持有对象就无需去管理对象 

---

## 02实用的内存管理方法

1.我们在开发过程中对于一个类的管理是很常见的,例如一个继承NSObject类 拥有属性值

```objc
@interface Counter : NSObject
@property (nonatomic, retain) NSNumber *count;
@end;

- (NSNumber *)count {
    return _count;
}
```
2.我们优先采用 访问器方法(即gettter)方法,即使它看起来繁琐,但是他可以将release retain 方法给避免掉,进而内存管理起来更方便
同理我们使用访问器去设置属性时,也可以采用官方推荐做法 去避免产生对象
```objc
//这种没有产生NSNumber对象
- (void)reset {
    NSNumber *zero = [NSNumber numberWithInteger:0];
    [self setCount:zero];
}

//下面这种就需要手动管理
- (void)reset {
    NSNumber *zero = [[NSNumber alloc] initWithInteger:0];
    [self setCount:zero];
    [zero release];
}
```

3.不要在initializer 和delloc中使用访问器方法
如果需要在init中调用 请操作如下 因为在init方法中 self初始化还未完成.
```objc
- init {
    self = [super init];
    if (self) {
        _count = [[NSNumber alloc] initWithInteger:0];
    }
    return self;
}
```


## 03 弱引用避免循环引用

retain是对一个对象的强引用,当一个对象的强引用被释放的条件是当前没有人再引用它.当A -> B且 B -> A时候 彼此被对方强引用,导致满足释放的条件,此时就会造成内存泄漏.
![](/images/e9a0f8bbb2e81b13e4408cc8e68a397a.shentu.webp)
为了解决循环引用我们需要引入 如弱引用 weakReference 在cocoa中规定:**parent客体对child保持强引用,但是child对parent应具有弱引用**
- data source (数据源)
- outline view item (view)
- observers(通知的观察者)
- miscellaneous targets and delegates(目标和代理)
  
## 04避免对象被释放

1.从数组或者字典中移除的对象
```objc
//当对象被移除之后 heisenObject为空了
heisenObject = [array objectAtIndex:n];
[array removeObjectAtIndex:n];
heisenObject could now be invalid.
```

2.父对象被释放了

3.给一个已经释放的对象调用[object release],如1中的 [heisenobject release]之后

>使用autoreleasepool

autoreleasepool自动释放池 是可以循环嵌套的
```objc
@autoreleasepool {
    //创建自动释放对象的代码。
}✅

@autoreleasepool {
    //创建自动释放对象的代码。
    @autoreleasepool {
    //创建自动释放对象的代码。
  }
}✅
```
有三种情况下会使用到自动释放池
- 不基于UIkit开发的程序
- 创建很多的临时对象循环
- 生成第二个线程 (一旦线程看是执行 那么就必须创建自己的自动释放池块 否则就会出现内存泄漏的对象)


