+++
date = '2018-03-17T14:54:44+08:00'
draft = false
title = '二进制文件重排'
tags = ["binary","code","ios"]
+++
<!-- more -->

![lancher](/images/ww2.webp "start")

名称来自战争术语 抢滩登陆 (beach head)
目的是更快的将iOS的application启动提速. 让用户更快的打开app !推广一波🤣 [开源库](https://github.com/forwyt/beachhead)

## 进程访问内存的原理

> 在计算机中，**进程** 是操作系统分配给正在运行的程序的一种资源。它可以看作是程序的执行实例，包括程序计数器、寄存器、堆栈和打开的文件等。每个进程都在自己的**虚拟地址**空间中运行，这使得每个进程都有自己独立的内存空间。

### 虚拟地址
1. 地址空间划分：操作系统将每个进程的地址空间划分为多个固定大小的页。通常，每个页的大小为4KB
2. 页表创建：操作系统为每个进程创建一个页表，用于将虚拟地址映射到物理地址。页表是一个数据结构，它记录了每个虚拟页对应的物理页的地址。

3. 虚拟地址转换：当进程访问内存时，它使用虚拟地址。操作系统通过查找页表中的映射关系，将虚拟地址转换为物理地址。这个转换过程通常包括两个步骤。
- 页目录查找：进程的虚拟地址通常被划分为三个部分：页目录索引、页表索引和页内偏移。首先，操作系统使用页目录索引在页目录中查找对应的页表。
- 页表查找：操作系统使用页表索引在找到的页表中查找对应的物理页的地址
4. 内存访问：当虚拟地址被转换为物理地址后，进程可以使用物理地址来访问内存。这样，进程就可以在自己的虚拟地址空间中进行内存操作，而不需要关心物理内存的实际分布。 


为了提高效率和方便管理，又对虚拟内存和物理内存又进行分页（Page）。当进程访问一个虚拟内存 Page 而对应的物理内存却不存在时，会触发一次缺页中断（Page Fault），分配物理内存，有需要的话会从磁盘 **mmap** 读人数据。
mmap
> mmap是一种内存映射技术，它允许应用程序将文件或其他设备映射到自己的虚拟地址空间中，从而可以像访问内存一样访问这些数据。mmap可以提供高效的读写文件的方式，并且还可以用于共享内存和匿名内存映射等场景


### Page Fault



> 当进程访问一个虚拟内存 Page 而对应的物理内存却不存在时，会触发一次缺页中断（Page Fault），分配物理内存，有需要的话会从磁盘 mmap 读人数据。通过 App Store 渠道分发的 App，Page Fault 还会进行签名验证，所以一次 Page Fault 的耗时比想象的多


1.虚拟内存允许操作系统摆脱物理 RAM 的限制。虚拟内存管理器创建一个逻辑地址空间（或“每个进程的“虚拟”地址空间）并将其划分为大小统一的内存块，称为页面。处理器及其内存管理单元（MMU）维护一个页表将程序逻辑地址空间中的页面映射到计算机 RAM 中的硬件地址。当程序代码访问内存中的地址时，MMU 使用页表将指定的逻辑地址转换为实际的硬件内存地址。这种转换是自动发生的，并且对于正在运行的应用程序是透明的。

2.对于程序而言，其逻辑地址空间中的地址始终可用。但是，如果应用程序访问当前不在物理 RAM 中的内存页上的地址，发生页面错误。当这种情况发生时，虚拟内存系统会调用一个特殊的页面错误处理程序来立即响应该错误。页面错误处理程序停止当前正在执行的代码，找到物理内存的空闲页面，从磁盘加载包含所需数据的页面，更新页表，然后将控制权返回给程序代码，程序代码可以访问内存地址通常情况下。这个过程被称为寻呼。

3.如果物理内存中没有可用的空闲页面，则处理程序必须首先释放现有页面以为新页面腾出空间。系统如何发布页面取决于平台。在 OS X 中，虚拟内存系统经常将页面写入后备存储。这后备存储是基于磁盘的存储库，其中包含给定进程使用的内存页的副本。将数据从物理内存移动到后备存储称为调出（或“交换出”）；将数据从后备存储移回物理内存称为分页（或“交换”）。在 iOS 中，没有后备存储，因此页面永远不会调出到磁盘，但只读页面仍会根据需要从磁盘调入。

4.在 OS X 和早期版本的 iOS 中，页面的大小是 4 KB。在更高版本的 iOS 中，基于 A7 和 A8 的系统向由 4 KB 物理页支持的 64 位用户空间公开 16 KB 页面，而 A9 系统公开由 16 KB 物理页支持的 16 KB 页面。这些大小决定了发生页面错误时系统从磁盘读取多少KB。当系统花费过多的时间处理页面错误以及读写页面而不是执行程序代码时，就会发生磁盘抖动。

任何类型的分页，尤其是磁盘抖动，都会对性能产生负面影响，因为它迫使系统花费大量时间读取和写入磁盘。从后备存储读取页面需要花费大量时间，并且比直接从 RAM 读取要慢得多。如果系统必须先将一页写入磁盘，然后才能从磁盘读取另一页，则性能影响会更严重。


### 重排
编译器在生成二进制代码的时候，默认按照链接的 Object File(.o)顺序写文件，按照 Object File 内部的函数顺序写函数。

page1 与 page2 都需要从初始化状态到加载到物理内存中，这样如上图会触发两次 Page Fault。 二进制重排 的做法就是将 function1 与 function2 放到一个内存页中， 那么启动时则只需要加载一次 page 即可

![image.png](/images/6cb8617b4f56968e9b8fe2576ded439f.image.webp)

以Demo为例:`(路径在$(TARGET_TEMP_DIR)/$(PRODUCT_NAME)-LinkMap-$(CURRENT_VARIANT)-$(CURRENT_ARCH).txt)`

```objc
**0x1000020F0    0x000000A0    [  2] -[BHAppDelegate application:didFinishLaunchingWithOptions:]
0x100002190    0x00000070    [  2] -[BHAppDelegate applicationWillResignActive:]
0x100002200    0x00000070    [  2] -[BHAppDelegate applicationDidEnterBackground:]
0x100002270    0x00000070    [  2] -[BHAppDelegate applicationWillEnterForeground:]
0x1000022E0    0x00000070    [  2] -[BHAppDelegate applicationDidBecomeActive:]
0x100002350    0x00000070    [  2] -[BHAppDelegate applicationWillTerminate:]
0x1000023C0    0x00000050    [  2] -[BHAppDelegate window]
0x100002410    0x00000060    [  2] -[BHAppDelegate setWindow:]
0x100002470    0x00000050    [  2] -[BHAppDelegate .cxx_destruct]
0x1000024C0    0x0000001C    [  2] _sancov.module_ctor_trace_pc_guard
0x1000024E0    0x00000060    [  3] -[BHViewController viewDidLoad]
0x100002540    0x00000060    [  3] -[BHViewController didReceiveMemoryWarning]
0x1000025A0    0x00000050    [  3] -[BHViewController otherFucntionBeforTargetFuction1]
0x1000025F0    0x00000050    [  3] -[BHViewController otherFucntionBeforTargetFuction2]
0x100002640    0x000000D0    [  3] -[BHViewController createOrderClick:]
0x100002710    0x0000001C    [  3] _sancov.module_ctor_trace_pc_guard
0x100002730    0x000000B0    [  4] _main
**
```
可以关注此处代码 即使在执行时优先级 function2 > funtion1 但是在ld时 是按照定义的顺序来插入即 funtion 1 > function2. 明白这个原理之后我们可以通过clang-插装的原理理清楚程序执行的真实循序来给二进制文件做排序. 尽可能多的将函数调用顺序相近的function排列在同一张Page里面.减少page faut 触发的页断次数.从而达到减少 启动时间.

```objc
-(void)otherFucntionBeforTargetFuction1{
    NSLog(@"just for test");
}

-(void)otherFucntionBeforTargetFuction2{
    NSLog(@"just for test");
}

- (IBAction)createOrderClick:(id)sender {
    NSLog(@"create order click");
    [BeachAnalyze version];
    [self otherFucntionBeforTargetFuction2];
    [self otherFucntionBeforTargetFuction1];
}

```
修改之后的顺序如下：

```objc
_main
-[BHAppDelegate setWindow:]
-[BHAppDelegate application:didFinishLaunchingWithOptions:]
-[BHViewController viewDidLoad]
-[BHAppDelegate applicationDidBecomeActive:]
-[BHAppDelegate window]
-[BHViewController createOrderClick:]
-[BHViewController otherFucntionBeforTargetFuction2]
-[BHViewController otherFucntionBeforTargetFuction1]

```
操作流程
集成SDK pod 'beachhead', :git => 'https://github.com/forwyt/beachhead.git',:tag => '0.1.0'

修改other c flag 在 工程的 taget -> build setting -> other c flag 插人下列参数-fsanitize-coverage=func,trace-pc-guard

调用api 仅需一行代码 在您想结束启动时间统计的业务中插入, 类似在MainViewController 中 viewDidAppear 为结束时间点.  [BeachAnalyze getOrderFile];

观察打印的日志  重排之后的执行顺序 🚀 : /xx/xx/xx/data/Containers/Data/Application/93BDE7B1-28AF-4203-A43C-DA5D71709D38/tmp/BeachHead.order

将重新排列之后的order利用 将路径对应的文件copy到项目根目录 然后将 工程的 taget -> build setting -> linking -> order file 写入刚才的路径 即可以完成二进制重排 

### Tips
- Page Fault，开销在 0.6 ~ 0.8ms。 当进程在进行一些计算时，CPU 会请求内存中存储的数据。在这个请求过程中，CPU 发出的地址是逻辑地址（虚拟地址），然后交由 CPU 当中的 MMU 单元进行内存寻址，找到实际物理内存上的内容。若是目标虚存空间中的内存页（因为某种原因），在物理内存中没有对应的页帧，那么 CPU 就无法获取数据。这种情况下，CPU 是无法进行计算的，于是它就会报告一个缺页错误（Page Fault）。因为 CPU 无法继续进行进程请求的计算，并报告了缺页错误，用户进程必然就中断了。这样的中断称之为缺页中断。在报告 Page Fault 之后，进程会从用户态切换到系统态，交由操作系统内核的 Page Fault Handler 处理缺页错误。
- 如果操作有疑问请 参照Demo文件夹 包含了引入SDK到缠住order配置.

### 灵感来源

基于抖音二进制重排优化一文
clang 插桩