# Rust 入门学习

## 环境安装

使用 rustup 安装 rust 开发环境

```bash

curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

```

安装完成后按照提示配置环境变量，然后查看版本号

```bash

rustc --version

```

## Cargo 使用

官方 [examples](https://doc.rust-lang.org/stable/rust-by-example/index.html) 提供了 Cargo 的使用

创建工程

```bash

cargo new foo

```

工程组织方式如下：

```bash

foo
├── Cargo.toml
└── src
    └── main.rs

```

## 依赖管理

```toml

[package]
name = "foo"
version = "0.1.0"
authors = ["mark"]

[dependencies]
clap = "2.27.1" # from crates.io
rand = { git = "https://github.com/rust-lang-nursery/rand" } # from online repo
bar = { path = "../bar" } # from a path in the local filesystem

```

- crates.io ： rust 的依赖仓库
- git： 在线依赖
- local： 本地依赖

## 国内仓库

编译 $HOME/.cargo/config 配置文件，没有则新建

```config

[source.crates-io]
replace-with = 'ustc'

[source.ustc]
registry = "https://mirrors.ustc.edu.cn/crates.io-index"

```

参考链接：http://mirrors.ustc.edu.cn/help/crates.io-index.html

[The Rust Programming Language](https://doc.rust-lang.org/book/title-page.html)

[rust 程序设计语言中文](https://kaisery.github.io/trpl-zh-cn/title-page.html)