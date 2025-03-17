+++
date = '2016-09-12T14:44:52+08:00'
draft = false
title = 'ios内存原理（2）'
tags = ["binary","code","ios"]
+++

![](/images/9fb61babf6f061ea4929598c55cd8c2e.programming-1873854_1920.webp)
经过前面文档我们大致了解了arc/mrc机制,是为了更好的去做iOS的内存管理
<!-- more -->

>前言
>
经过前面文档我们大致了解了arc/mrc机制,是为了更好的去做iOS的内存管理.在面对一个对象的时候,从初始化到delloc销毁的过程中进行合适的插入 *retain* 和 *release* 做到内存的回收释放♻️,随着iOS的迭代发展,我们开始逐渐采用一种解放程序员的新型管理模式 ARC ...

### 什么是ARC/MRC

**Manual Reference Counting** 是根据对象的引用计数器 **reference count** 来判断在何时将一个对象所占用的内存回收
**Automatic Reference Counting** 自动引用计数 是苹果子WWDC2011年引入的,其工作原理下面详细理解,根本上是通过强指针来判断

### AutoreleasePool

>从main函数开始了解:

```objc
int main(int argc, const char * argv[]) {
   @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```
在这个autoreleasepool的block里面 所有的事件,消息,全部转交给了**UIApplication**来处理,从此可以看出一个iOS应用是完全包裹在自动释放池里面的

通过clang 编译器将main函数转换成了 main.cpp

```objc
   $ clang -rewrite-objc main.m
```
我们将关注点放在__AtAutoreleasePool这个结构体上面
```cpp
struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
};
```

不难看出我们main函数的工作原理大致上是在初始化时候调用了
```
void * atautoreleasepoolobj = objc_autoreleasePoolPush();
```
然后在析构时候调用了

```
objc_autoreleasePoolPop(atautoreleasepoolobj);```

所以@autoreleasepool会被转换成上面两行代码

> objc_autoreleasePoolPush

从[源码](https://opensource.apple.com/source/objc4/objc4-532/runtime/NSObject.mm.auto.html "标题")中我们看到

```objc
void *_objc_autoreleasePoolPush(void) { return objc_autoreleasePoolPush(); }
void _objc_autoreleasePoolPop(void *ctxt) { objc_autoreleasePoolPop(ctxt); }

void *
objc_autoreleasePoolPush(void)
{
    if (UseGC) return NULL;
    return AutoreleasePoolPage::push();
}

void
objc_autoreleasePoolPop(void *ctxt)
{
    if (UseGC) return;

    // fixme rdar://9167170
    if (!ctxt) return;

    AutoreleasePoolPage::pop(ctxt);
}
```

push和pop是操作这一个 **AutoreleasePoolPage** 对象

> AutoreleasePoolPage

```objc
class AutoreleasePoolPage 
{

#define POOL_SENTINEL 0
    static pthread_key_t const key = AUTORELEASE_POOL_KEY;
    static uint8_t const SCRIBBLE = 0xA3;  // 0xA3A3A3A3 after releasing
    static size_t const SIZE = 
#if PROTECT_AUTORELEASEPOOL
        4096;  // must be multiple of vm page size
#else
        4096;  // size and alignment, power of 2
#endif
    static size_t const COUNT = SIZE / sizeof(id);

    magic_t const magic;
    id *next;
    pthread_t const thread;
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t hiwat;

    // SIZE-sizeof(*this) bytes of contents follow

    static void * operator new(size_t size) {
        return malloc_zone_memalign(malloc_default_zone(), SIZE, SIZE);
    }
    static void operator delete(void * p) {
        return free(p);
    }

    inline void protect() {
#if PROTECT_AUTORELEASEPOOL
        mprotect(this, SIZE, PROT_READ);
        check();
#endif
    }

    inline void unprotect() {
#if PROTECT_AUTORELEASEPOOL
        check();
        mprotect(this, SIZE, PROT_READ | PROT_WRITE);
#endif
    }
   
```

从上面的源码简略分析一下 AutoreleasePoolPage 类 包含了
- magic 用于校验AutoreleasePoolPage的完整性
- thread 保存了page所在的线程
- size_t 常量4096字节大小 0X1000
-  parent 父指针
- child 子指针 通过父子指针来构造出一条双向链表
![](https://tva1.sinaimg.cn/large/007S8ZIlgy1ge8f1lj334j30ys08udia.jpg)

AutoreleasePoolPage 类是有56bit用来存储成员变量 剩余的都是用来存储加入自动释放池的对象 通过begin() end()
来帮我们快速获取到内存地址的边界
next()指向了下一个page的一片区域

>POOL_SENTINEL(哨兵对象)

#define POOL_SENTINEL nil
哨兵对象只是nil的一个别称,当每一次push()压栈的时候 都会把一个nil对象放在栈顶 并且会返回该哨兵对象

当调用pop()时候就会向释放池里面发送release消息 直到出现了第一个 POOL_SENTINEL 

```objc
//1.先调用push
void *objc_autoreleasePoolPush(void) {
    return AutoreleasePoolPage::push();
}
//2.push定义如下
static inline void *push() {
   return autoreleaseFast(POOL_SENTINEL);
}
//3.在这里会进入一个比较关键的方法 autoreleaseFast，并传入哨兵对象 POOL_SENTINEL
static inline id *autoreleaseFast(id obj)
{
   AutoreleasePoolPage *page = hotPage();
   if (page && !page->full()) {
       return page->add(obj);
   } else if (page) {
       return autoreleaseFullPage(obj, page);
   } else {
       return autoreleaseNoPage(obj);
   }
}
//4 将对象添加到自动释放池页中
id *add(id obj) {
    id *ret = next;
    *next = obj;
    next++;
    return ret;
}
```
上述的代码逻辑很简单就是 
1.pool 调用poolpage的push 
2.page 调用push()收到一个返回参数 POOL_SENTINEL 告诉我们该压入对象的栈顶内存地址  
3.fast 会根据pool_sentinel 的地址来计算
* 有hotPage 并且当前 page 不满调用 page->add(obj) 方法将对象添加至 AutoreleasePoolPage 的栈中
* 有 hotPage 并且当前 page 已满调用 autoreleaseFullPage 初始化一个新的页调用 page->add(obj) 方法将对象添加至 AutoreleasePoolPage 的栈中
* 无 hotPage 调用 autoreleaseNoPage 创建一个 hotPage调用 page->add(obj) 方法将对象添加至 AutoreleasePoolPage 的栈中
4.最终的执行代码都是page去调用了一个 add()方法

最终我们流程跑完了总结一个 方法的调用栈
```oc
[NSObject autorelease]
└── id objc_object::rootAutorelease()
    └── id objc_object::rootAutorelease2()
        └── static id AutoreleasePoolPage::autorelease(id obj)
            └── static id AutoreleasePoolPage::autoreleaseFast(id obj)
                ├── id *add(id obj)
                ├── static id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page)
                │   ├── AutoreleasePoolPage(AutoreleasePoolPage *newParent)
                │   └── id *add(id obj)
                └── static id *autoreleaseNoPage(id obj)
                    ├── AutoreleasePoolPage(AutoreleasePoolPage *newParent)
                    └── id *add(id obj)
```