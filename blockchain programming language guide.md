<h1>区块链编程语言学习指南</h1>


# 前言

  &emsp;&emsp;在全面开发指南有讲到，区块链开发可以划分为三大模块

## 基础设施模块

### devops

  &emsp;&emsp;主要工作包括区块链部署运维等，需要掌握相关工具的工具使用，涉及到python,shell，go等。

### 底层链

  &emsp;&emsp;底层链包括公链，侧链，跨链等开发技术，涉及go，rust，java，c++，nodejs等语言，对于区块链链而言，是不限制语言的。不过，比如主流的以太坊全节点客户端Geth是Go语言实现，而波卡生态系基本是rust。

  建议重点掌握go语言和rust语言。下面会以这两种语言作为学习指南。

## 智能合约模块

  &emsp;&emsp;指南合约方面，现在主流的是以太坊智能合约开发，主要的开发语言是solidity，不过也支持其它语言，不过目前流行度没那高。其它底层公链，比如bsc，heco也也兼容solidity开发的智能合约。

## 前端交互模块

  &emsp;&emsp;前端交互模块，主要了解h5前端，Android，IOS等。主要涉及语言javascript，java，swift。这部分可以了解，不需要重点研究。

# go语言

  &emsp;&emsp;Go（又称 Golang）是 Google 开发的一种静态强类型、编译型语言。Go 语言语法与 C 相近，但功能上有：内存安全，GC（垃圾回收），并发计算等。

  &emsp;&emsp;学习总结：重点研究go语言的协程，通道，垃圾回收，内存分配，同步机制，cgo以及语言特性等。

  学习资料参考：

  [Go语言圣经 《The Go Programming Language》 中文版本](https://docs.hacknode.org/gopl-zh/index.html)

  [Go 语言设计与实现](https://draveness.me/golang/)

  [Go进阶训练营](https://u.geekbang.org/subject/go?utm_source=time_web&utm_medium=menu&utm_term=timewebmenu&utm_identify=geektime&utm_content=menu&utm_campaign=timewebmenu&gk_cus_user_wechat=university)

  [Go并发编程 — sync.Once 单实例模式的思考](https://zhuanlan.zhihu.com/p/357952785)

# rust语言

   &emsp;&emsp;Rust是类似于c语言的开发语言，和Go语言也比较接近，目前在Android系统层开发也引入了Rust语言的支持。在区块链项目中大量用到了Rust语言。比如filecoin以及波卡生态。

   [Rust官方文档](https://www.rust-lang.org/zh-CN/tools/install)

   [Rust 程序设计语言](https://kaisery.github.io/trpl-zh-cn/title-page.html)

# solidity语言

# java语言
   
  java开发语言在区块链开发生态相对少些，但是也有很多其实现。

  参考：

  [java编程思想](https://blog.didispace.com/books/think-in-java/)

  [Effective Java 2nd Edition](https://github.com/HugoMatilla/Effective-JAVA-Summary)

# python语言

  [Python3 CookBook中文版](https://www.kancloud.cn/kancloud/python3-cookbook/47412)

# c++

大部分矿池都是C++实现的，也有不少底层链公链用C++编写。

参考：

  [Effective C++](https://www.kancloud.cn/wizardforcel/effective-cpp)
  [Hunter: organize freedom](https://hunter.readthedocs.io/en/latest/)






