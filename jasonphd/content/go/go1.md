+++
date = '2017-01-20T09:50:48+08:00'
draft = true
title = 'Go1'
tags = ["code","go"]

+++
![](/images/c4673178de45c376576bb1748eff5017.go.webp)
go的一些入门

# Go Advantages
> Go advantages
- code run fast
- garbage collection
- simpler objects
- concurrency is efficient (并发性好)


language
* machine language
* assembly language
* high-level language

Compiled 编译型语言 类似C C++ JAVA
 
Interpretation 解释型语言 类似 Python JAVA,当执行的时候回将源代码翻译为机器执行的代码 所以会消耗时间


> Object

虽然在go语言中 没有意义上的对象.但是struck 充当了类似的角色

> Concurrency并发
- 如果在没有高并发会遇到
    - Moore law 18个月的晶体管会翻倍 硬件性能增长收到限制
    - 功耗会增加 
- parallelism并行

> variables
命名规则
- keyword避免掉
- declarations 申明变量 var x int  
    - var 是keyword
    - x 是变量的名称
    - int 是类型
- 可以多行命名变量 var x,y,z int
- 类型
   + Integer (only integer value )
   + Floating Point (Fractional decimal value, may use different with hardware)
   + String (seqyebces characters)

- Init初始化变量
    + type IDNumber int
     var x int = 100 / var x = 100都是可以 有自动推断类型
    + **当为赋值的初始化时候 会有zero valu (类似 0 "" )**
    + 简化赋值 x := 100 这种声明和操作一起进行分配只能在函数内部使用, 不可以在函数外进行简短的变量声明
    ```go
      package main
      import (
      	"fmt"
      )
      var y = 1000
      
      func main()  {
      	x := 100
      	print(x )
      	print(y)
      	fmt.Print("hello go course\n1");
      
      }
    ```

> pointers

详细的定义
 - a ponter is an address to data in memory
 - **&** oprrator returns the address of a variable / function
 讲该变量或者函数的的内存地址返回
 - <big>*</big>  operator return data at an address 
 讲的是在返回值
 - new()也是一种初始化
```go
	//定义一个初始变量x并赋值
	var x int = 100
	fmt.Print(&x);
	//定义一个ip变量 为int指针类型
	var ip *int
	//将ip指向x 也就是ip = 一个内存地址(类似0xc000016080)
	ip = &x;
	//那么在ip签名加上*
	*ip = * &x (所以就是一个指向内存地址的指针 所以*p = x)
```
   


> variable scope / 变量的域

- Block
    + a sequence of declarations and statements within matching brackets { }
    + Universe block : all Go Source 
    + Package block - all source in a package
    + File block - all source in a file 


![](https://jasonphd.com/pic/go_learning01.png)
 
> Deallocating Space / 内存释放

当一个变量不在被需要的时候那么它应该被销毁

- 例如在一个func 里面 有一个变量X 每次使用结束后x会被释放 它不会被引用着 

># Stack & Heap

STACK:
+ stack is dedicated to fuction call 
+ lcoal variables are stored here 
+ deallocated after function completes

HEAP
+ heap is persistent

MANUAL Deallocation


> Go Garbage Collection / Go的垃圾回收 ♻

in interpreted languages, this is done by the interpreter
- compiler determines stack / heap
- garbage collection is the background 





# code

> comment 
- // singl-linecomments
- block comment /* xxxx */  


###**占位符**

|  占位符   | 值  |
|  ----  | ----  |
| %v  | 默认的格式 |
| %T  | 值类型 |
| %t  | 布尔值 |
| %b  | 二进制 |
| %c  | Unicode |
| %d  | 十进制 |
| %o  | 八进制 |
| %f  | 浮点值 |
| %s   | string或者是[ ]byte |
| %p  | 指针 |

> 整数 浮点数 字符串
隐式转换
- convert type with T()
Float
- float32 ~6 digits of precision(精度)
- float64 ~15 digits of precision
 ASCII & Unicode
 
>一、ASCII 码
我们知道，计算机内部，所有信息最终都是一个二进制值。每一个二进制位（bit）有0和1两种状态，因此八个二进制位就可以组合出256种状态，这被称为一个字节（byte）。也就是说，一个字节一共可以用来表示256种不同的状态，每一个状态对应一个符号，就是256个符号，从00000000到11111111。
上个世纪60年代，美国制定了一套字符编码，对英语字符与二进制位之间的关系，做了统一规定。这被称为 ASCII 码，一直沿用至今。
ASCII 码一共规定了128个字符的编码，比如空格SPACE是32（二进制00100000），大写的字母A是65（二进制01000001）。这128个符号（包括32个不能打印出来的控制符号），只占用了一个字节的后面7位，最前面的一位统一规定为0。

>二、非 ASCII 编码英语用128个符号编码就够了，但是用来表示其他语言，128个符号是不够的。比如，在法语中，字母上方有注音符号，它就无法用 ASCII 码表示。于是，一些欧洲国家就决定，利用字节中闲置的最高位编入新的符号。比如，法语中的é的编码为130（二进制10000010）。这样一来，这些欧洲国家使用的编码体系，可以表示最多256个符号。
但是，这里又出现了新的问题。不同的国家有不同的字母，因此，哪怕它们都使用256个符号的编码方式，代表的字母却不一样。比如，130在法语编码中代表了é，在希伯来语编码中却代表了字母Gimel (ג)，在俄语编码中又会代表另一个符号。但是不管怎样，所有这些编码方式中，0--127表示的符号是一样的，不一样的只是128--255的这一段。
至于亚洲国家的文字，使用的符号就更多了，汉字就多达10万左右。一个字节只能表示256种符号，肯定是不够的，就必须使用多个字节表达一个符号。比如，简体中文常见的编码方式是 GB2312，使用两个字节表示一个汉字，所以理论上最多可以表示 256 x 256 = 65536 个符号。
中文编码的问题需要专文讨论，这篇笔记不涉及。这里只指出，虽然都是用多个字节表示一个符号，但是GB类的汉字编码与后文的 Unicode 和 UTF-8 是毫无关系的。

>三. Unicode正如上一节所说，世界上存在着多种编码方式，同一个二进制数字可以被解释成不同的符号。因此，要想打开一个文本文件，就必须知道它的编码方式，否则用错误的编码方式解读，就会出现乱码。为什么电子邮件常常出现乱码？就是因为发信人和收信人使用的编码方式不一样。
可以想象，如果有一种编码，将世界上所有的符号都纳入其中。每一个符号都给予一个独一无二的编码，那么乱码问题就会消失。这就是 Unicode，就像它的名字都表示的，这是一种所有符号的编码。
Unicode 当然是一个很大的集合，现在的规模可以容纳100多万个符号。每个符号的编码都不一样，比如，U+0639表示阿拉伯字母Ain，U+0041表示英语的大写字母A，U+4E25表示汉字严。具体的符号对应表，可以查询unicode.org，或者专门的汉字对应表。

此处插入两条讲解非常好的老师blog

[阮一峰](http://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html)

[JOEL](https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/)
# 进阶数据类型

###Arrays / 数组
基本特性
- fixed-lenth series of elements of a choosen type
- element accessd using subscript notation []
- indices start at 0
- elements initialized to zero value 
####array literal(文字型数组)
- x := [...]{1,2,1,2} ...表示不确定elementsd的个数

###Slices / 切片
- a window on an uderlying array
- variable size up to the whole array
- 必备的三个元素
    + poniter indicastes the smart of the slices
    + length is the number of elts in the slices
    + capacity is maximum number of elts 
```go
arr := [...]string{"a","b","c","d","e","f"}

s1 := arr [1:3] // b c 
s2 := arr [2:5] // c d e
左边包含 右边不包含

lenth 表示当前切片的长度
capacity 表示切片可以拓展的最大长度 表示就是 数组的lenth

```
切片和数组的区别:
+ sli := []int{1,2,3} 切片
+ sli := [...]int{1,2,3} 数组


###variable slices / 可变切片
####make() 函数
```go
//两个参数的make调用时候 第一个传入的事数组的类型 第二个是长度
 sli1 = make(int[],10)
//三个参数的时候 第三个参数表示的是 数组的最大长度 其实也是切片的容量 capatity
 sli2 = make(int[],10,15)        

```
###Append() / 添加

- size of a slice can be increased by append()
- adds elements to the end of a slice
- inserts into underlying array
- increases size of array if necessary


### Hash Table 

- container key /value pairs
- each key is associated with a unique key 
- Hash Fuction is used to compute the slot for a key 

Tradeoffs of Hash Tables
优点
- faster lookup than lists 
- arbitrary keys
缺点 
- may have collisions

🚀 **拓展**



### Map

implementation of a hash table 
```go
var idMap map [string]int
```
- referencing a value with [key]
- return zero if key is not present
- adding a key/value pair

iteraring through a map
- use a loop with the range keyword
- two-value assignmeny with range


### Struct / 结构体

- aggregate data type
- groups together other objects of arbitrary type
定义
```go
type struct Person{
    name string
    address string
    phone string 
}
var p Person
```





