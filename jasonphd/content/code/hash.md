+++
date = '2018-03-20T04:40:22+08:00'
draft = true
title = 'hash的一些记录'
tags = ["code"]
+++

![](/images/eb20681b7a24e34a8856bfa8b034cb0d.xray.webp)
当打开Node或者任何一个开源平台下载代码时候,我们都会很在意该网站是不是属于官方认证...
<!-- more -->

## Hash 哈希的理解与运用


当打开Node或者任何一个开源平台下载代码时候,我们都会很在意该网站是不是属于官方认证.
但是即使是官网下载的文件,在传输过程中被劫持.那么我们如何保证自己下载的app或者说源代码又或是zip是属于没有被篡改的呢?
这个时候你会看到 会有一个校验的码.





首先,要想验证文件的完整性和安全性,是基于对该文件的一些数据做算法的摘要的比对.
我们来做一个小测试
```shell script
$ shasum xray.jpg                                          
501575ac28e10b1632e4ca5d7d69737ce5c532c1  xray.jpg
```
调出终端 通过Mac osx 自带的命令 **shasum**,
你可以对一个文件做一个sha算法摘要 
<table><tr><td bgcolor=orange>501575ac28e10b1632e4ca5d7d69737ce5c532c1</td></tr></table>
然后我们打开xray.jpg 做一下简单的调整颜色明度 再次做shasum命令后你会发现
<table><tr><td bgcolor=orange>91356a873b5b570b494faee59018a1de83769b1d</td></tr></table>
这时候 xray.jpg 图片即使在文件大小或者说文件名一致的情况下 哈希值变了. 那么我们就可以合理的推断出 
hash值可以用来做文件的校验.

## 什么是hash fuction

>哈希函数是一种将可变长度的输入映射到固定长度的输出的算法。该函数的返回值称为哈希值，摘要或哈希。
 它主要用于解决密码学的完整性原理。该消息可以在发送方和接收方之间的通信期间进行更改

哈希函数的主要特征是：
- 定长输出：散列函数接收任何大小的消息（输入），并始终产生相同的输出大小。
- 效率：一定不会很难执行。
- 确定性：同一条消息将始终产生相同的哈希值。

要在密码学中使用，它必须具有以下属性：
- 图像前抗性：给定哈希值，很难找到源于该哈希值的消息。
- 第二种图像前抗性：给定一条消息m，要找到另一个产生与m相同哈希值的消息n会非常困难。
- 防冲突性：很难找到产生相同哈希值的两条不同消息。

哈希值的利用举例
- 文档完整性
- 密码存储 因为在高级的程序设计中我们可以根据哈希值得唯一性以及可逆推断差的特性来将明文密码转变为加密的密码
- 唯一的ID 例如 UUID/IDFA/AndroidID 这类代表用户信息唯一的,或比如我们每一次做的 git提交的时候 提交的内容


采用go语言做的 hash 实践 
```go
package main

import (
	"crypto/md5"
	"crypto/sha1"
	"crypto/sha256"
	"fmt"
)

func main() {
	s := "Foo"

	hmd5 := md5.Sum([]byte(s))
	hsha1 := sha1.Sum([]byte(s))
	hsha2 := sha256.Sum256([]byte(s))

	fmt.Printf("   MD5: %x\n", hmd5)
	fmt.Printf("  SHA1: %x\n", hsha1)
	fmt.Printf("SHA256: %x\n", hsha2)
}
/**
MD5：1356c67d7ad1638d816bfb822dd2c25d 
SHA1：201a6b3053cc1422d2c3670b62616221d2290929 
SHA256：1cbec737f863e4922cee63cc2ebbfaafcd1cff8b790d8cfd2e6a5d550b648afa
*/

```
