# Rust 笔记 
此笔记只是记录在使用rust遇到的一些坑或者一些比较好用的工具或者其他的东东。本人主要用rust开发网络相关的程序或者桌面程序，因此不涉及音视频和其他的领域。

## 好用的工具

### 工程相关
[cross](https://github.com/cross-rs/cross):跨平台编译，甚至在github actions也可以直接用[cross](https://github.com/houseabsolute/actions-rust-cross)。

[tarpaulin](https://github.com/xd009642/tarpaulin):单元测试覆盖率。

[min-sized-rust](https://github.com/johnthagen/min-sized-rust):减少编译后的二进制文件大小。

### 网络编程相关
常用的tokio异步运行时不多说了。http web框架可能有争议,比如warp,actix-web可能性能会好一点，但是axum写起来更简单。sqlx虽然直接写sql，相信我，其他的ORM框架更难用！

[tracing](https://github.com/tokio-rs/tracing):tokio的日志模块，非常强大，在tokio运行时中强烈建议使用tracing配合tracing-appender做日志。

[hyper](https://github.com/hyperium/hyper):开发底层http1、http2必备的crates。

[axum](https://github.com/tokio-rs/axum):http web框架,CRUD boy必备。

[sqlx](https://github.com/launchbadge/sqlx):sqlx框架,CRUD boy 必备。

[reqwest](https://github.com/seanmonstar/reqwest):http client。

[clap](https://github.com/clap-rs/clap):开发终端工具/命令行程序必备。

[openssl](https://docs.rs/openssl/latest/openssl/):openssl相关，由于很多crate是基于openssl的，一般都是作为vendor引入编译的时候直接一起编译。

[nom](https://github.com/rust-bakery/nom):nom是一个开源的语法解析器。如果要做json/sql/xml/yaml解析，可以用nom   。

## 一些常见的问题和反直觉
### 一些常见的问题
#### 我不会生命周期，看了'a,'static,&'static,T: 'a 头疼怎么办？
如果用rust做web程序，根本用不到生命周期。我至今还不懂生命周期，照样用rust开发web，技术栈就是axum+tracing+sqlx+reqwest。

#### hyper做server会不会内存泄漏?
先说答案，默认的allocator，在并发数升高后会导致内存飙升，且并发数降低时，内存不会降低。当前的解决方案是使用MiMalloc(本人已经验证过，效果良好)。具体的问题讨论在[这里](https://github.com/hyperium/hyper/issues/1790)。


### 反直觉
#### 多线程通信我直接用Arc::Mutex是不是太Low了，使用RwLock或者[dashmap](https://github.com/xacrimon/dashmap)更好一点？
- 如果你肯定你的锁是读超多写很少的场景，那么直接用RwLock。
- 如果你不确定你的锁的使用场景，请使用互斥锁。

互斥锁并不慢，我使用互斥锁来存储redis的所有数据，最后多线程的[压测结果](https://github.com/lsk569937453/rcache)，rcache的吞吐量是redis的两倍。

#### 异步运行时一定优于同步吗?异步运行时一定快吗？
先说结果，不一定，而且异步的程序相比同步的程序肯定要慢的。

什么时候使用异步运行时开发呢？
- 网络编程就是一个非常好的例子，网络程序中，程序大量的时间都在读取IO，因此可以用异步获取超高的吞吐量。

什么时候使用同步运行时开发程序呢?

- 算法大赛，需要你写一个rust算法比比谁的算法快。
- [1brc](https://github.com/gunnarmorling/1brc/discussions/57)

####  tokio::sync::Mutex性能很低。
常用场景:我看tokio里面有Mutex我就直接用了，没想那么多。

这个Mutex的性能很低，相比std::sync::Mutex性能低一倍以上。如果对性能敏感的应用可以直接在异步运行时中使用std::sync::Mutex。感兴趣的可以看一下tokio的关于Mutex的[文档](https://github.com/tokio-rs/tokio/blob/tokio-1.38.0/tokio/src/sync/mutex.rs#L24)。压测的程序在[这里](https://rustcc.cn/article?id=7841ed4a-fb33-4558-aa00-f45890a4f3eb).但是在异步运行时中使用std的Mutex时需要注意的原则是“锁保护的代码段不跨越await”。

#### 收集到的释放锁的方式。
- 使用drop关键字释放锁
- a.lock().clone 也可以释放锁
- 使用大括号将加锁的代码括起来，代码执行出大括号则自动释放锁

#### 我看future提供了很多方法。这两种写法有性能差别吗？
```
let a=b.await?;
let c=a.read().await?;
```
```
let c=b.and_then(|d|d.read()).await?
```
看上去第二种写法只await一次，其实如上两种写法性能上没有本质差别,第二种写法只是使用了future这个crates提供的内部的状态机。