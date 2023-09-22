---
title: 通过docker编译rust
date: 2023-09-22 14:43:08
categories: ['rust']
tags: ['rust', 'docker']
---

在windows开发后，想编译后在linux里面运行。测试通过docker编译后看能否可以在linux里面运行。
linux环境使用的是wsl1 Debian，
创建一个普通rust文件demo2.rs：
```rust
fn main() {
    println!("Hello, world!");
    let num = demo();
    println!("{}", num)
}
fn demo() -> i32 {
    return 1;
}
```
<!-- more -->
### 普通编译
下载rust镜像
```shell
docker pull rust:1.72.1
```
进入容器后手动编译
```shell
# 进入容器
docker run -v ${PWD}:/usr/src/myapp --rm -it --entrypoint bash rust:1.72.1
cd /usr/src/myapp
# 编译
rustc demo2.rs
# 运行
./demo2
```
编译后正常运行。

之后通过wsl运行main提示`./demo2: /lib/x86_64-linux-gnu/libc.so.6: version 'GLIBC_2.33' not found (required by ./demo2)`

### 通过alpine镜像编译
下载镜像
```shell
docker pull rust:1.72.1-alpine3.18
```
> alpine 使用的是 musl libc

```shell
# 进入容器
docker run -v ${PWD}:/usr/src/myapp --rm -it --entrypoint sh rust:1.72.1
cd /usr/src/myapp
# 编译
rustc demo2.rs
# 运行
./demo2
```
编译后正常运行。
然后通过wsl编译后正常运行。

### 添加tokio后编译（alpine编译失败）
使用cargo创建一个项目。添加tokio依赖
Cargo.toml文件
```toml
tokio = {version="1.32.0", features = ["full"]}
```
main.rs文件
```rust
#[tokio::main]
async fn main() {
    println!("Hello, world!");
    let num = demo().await;
    println!("{}", num)
}
async fn demo() -> i32 {
    return 1;
}
```
编译项目
```shell
docker run --rm  -v ${PWD}:/usr/src/myapp -w /usr/src/myapp rust:1.72.1-alpine3.18 cargo build --release
```
使用`alpine`镜像编译过程中`tokio-macros`编译失败，提示失败`x86_64-alpine-linux-musl/bin/ld: cannot find crti.o: No such file or directory`。

进入容器
执行命令`find /usr/ -name crti*`找到文件，添加环境变量
```shell
LIBRARY_PATH=/usr/local/rustup/toolchains/1.72.1-x86_64-unknown-linux-musl/lib/rustlib/x86_64-unknown-linux-musl/lib/self-contained/:$LIBRARY_PATH 
export LIBRARY_PATH
```
还是编译失败，提示` (signal: 11, SIGSEGV: invalid memory reference)`等信息。
未找到解决办法。

### 添加tokio后编译（改为rust:1.72.1）
一样的编译流程
```shell
docker run --rm  -v ${PWD}:/usr/src/myapp -w /usr/src/myapp rust:1.72.1 cargo build --release
```
编译成功后，在wsl里面运行和之前出现一样的错误`./demo2: /lib/x86_64-linux-gnu/libc.so.6: version 'GLIBC_2.33' not found (required by ./demo2)`

进入容器
```shell
docker run -v ${PWD}:/usr/src/myapp --rm -it --entrypoint bash rust:1.72.1
cd /usr/src/myapp
rustup target add x86_64-unknown-linux-musl
cargo build --release --target=x86_64-unknown-linux-musl
```
编译成功，切换到wsl在`target/x86_64-unknown-linux-musl`执行编译后文件正常。

### 待处理
可以自己创建镜像然后添加命令`rustup target add x86_64-unknown-linux-musl`，生成自定义镜像，以免每次需要下载。

