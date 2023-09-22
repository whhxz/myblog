---
title: 初次使用rust
date: 2023-09-19 16:45:38
categories: ["rust"]
tags: ["rust"]
---

### 安装

直接使用`scoop`进行安装。为了图方便使用的是 gnu。

```powershell
scoop install rustup-gnu
```

安装好后通过`rustc --version`查看版本

<!-- more -->

### Hello World

```rust
fn main() {
    println!("Hello, world!");
}
```

编译运行

```powershell
rustc main.rs #编译
./main.exe #运行
```

### 开发工具

使用 VS Code 开发，插件使用`rust-analyzer`。

### 猜数字

官方第一个例子

> - https://doc.rust-lang.org/book/ch01-02-hello-world.html

使用 cargo 创建项目，cargo 属于 rust 构建和包管理工具。

```shell
cargo new --vcs=none cargo02
```

> --vcs=none 表示创建时不要版本管理

成品如下：

Cargo.toml

```toml
[package]
name = "cargo02"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
rand = "0.8.5"
```

main.rs

```rust
use std::{cmp::Ordering, io};

use rand::Rng;

fn main() {
    println!("猜数字!");
    let secret_number = rand::thread_rng().gen_range(1..=100);
    // println!("秘密数字是：{secret_number}");
    loop {
        println!("输入猜想数字.");
        let mut guess = String::new();
        io::stdin().read_line(&mut guess).expect("读取失败!");
        let guess: u32 = match guess.trim().parse() {
            Ok(num) => num,
            Err(_) => continue,
        };

        println!("数字：{guess}");
        match guess.cmp(&secret_number) {
            Ordering::Less => println!("太小"),
            Ordering::Greater => println!("太大"),
            Ordering::Equal => {
                println!("猜对了");
                break;
            }
        }
    }
}
```

cargo 相关命令：

`cargo build`：编译项目，编译后 exe 在`target/debug`目录下；如果在 toml 里面添加依赖后，也可以通过该命令下载依赖包。
`cargo run`：运行项目。
`cargo doc --open`：查看文档。

> * [官方文档 https://doc.rust-lang.org/stable/book](https://doc.rust-lang.org/stable/book/)