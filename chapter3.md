# 第三章 注册新的订阅者

- [第三章 注册新的订阅者](#第三章-注册新的订阅者)
  - [3.1 我们的策略](#31-我们的策略)
  - [3.2 选择一个 Web 框架](#32-选择一个-web-框架)
  - [3.3 我们的第一个端点：一个基本的健康检查](#33-我们的第一个端点一个基本的健康检查)
    - [3.3.1 上手 actix-web](#331-上手-actix-web)
    - [3.3.2 剖析一个 actix-web 应用](#332-剖析一个-actix-web-应用)
      - [3.3.2.1 Server - HttpServer](#3321-server---httpserver)
      - [3.3.2.2 Application - App](#3322-application---app)

上一章我们定义了我们将要打造的产品（一个邮件列表服务！），并将目光缩小到一组精确的需求。现在是撸起袖子开始干活的时候了。

本章将首先尝试实现该用例：
> 作为博客的访问者，
> 我希望订阅该邮件列表，
> 这样我就可以在博客更新时收到电子邮件通知。

我们希望我们的博客访问者在网页的嵌入式表单中输入他们的电子邮件地址。

该表单会触发一个 API 调用，后端服务器会实际处理相关信息，存储信息并返回一个响应。

本章将会专注于该后端服务器——我们将实现 POST `/subscriptions` 这一 API 端点。

## 3.1 我们的策略

我们正在从头开始一个新项目——有许多繁重的前期工作需要我们操心：

- 选择一个 web 框架并加以熟悉；
- 定义我们的测试策略；
- 选择一个和数据库交互的 crate （我们必须把电子邮件存储在某个地方！）；
- 定义数据库模式的更改机制（也称为数据库迁移）；
- 实际编写一些查询。

这些工作太多了，一头扎进去可能会让人不知所措。

为了让旅途更轻松一些，我们将增加一块垫脚石：在处理 `/subscriptions` 端点之前，我们先实现一个称为 `health_check` 的端点。这个端点没有任何业务逻辑，但却是一个熟悉 web 框架并了解它各个不同活动部件的好机会。

我们将全程依靠持续集成流水线执行检查——假如你还没有将它设置好，请快速查看第一章（或者直接获取一个[现成的模板](https://www.lpalmieri.com/posts/2020-06-06-zero-to-production-1-setup-toolchain-ides-ci/#5-2-ready-to-go-ci-pipelines)）。

## 3.2 选择一个 Web 框架

我们应该使用哪个 web 框架来编写我们的 Rust API 呢？

本节内容原本是要对比当今各个可用的 Rust web 框架的优缺点。但我最后发现这些内容太长了，将其嵌入到此处没有什么意义，因此我将其发表为一篇衍生文章：为了深入了解 `actix-web`、`rocket`、`tide` 和 `warp`，请查询《[Choosing a Rust web framework, 2020 edition](https://www.lpalmieri.com/posts/2020-07-04-choosing-a-rust-web-framework-2020-edition/)》。

简而言之，截至 2022 年 3 月，`actix-web`应该是你编写用于生产用途的 Rust APIs 的首选 web 框架——在过去几年里它被广泛地使用，它背后有一个庞大而健康的社区，它运行于 `tokio` 之上，因此也大大降低了不得不处理各个不同的异步运行时之间的不兼容性/互操作性的可能性。

尽管如此，`tide`、`rocket`和`warp`也有着巨大的潜力，在 2022 年晚些时候，我们最终可能做出与此前不同的决定——假如你在使用另一个 web 框架来跟随本书的内容，我将很高兴想要看看你的代码！请发送电子邮件给我，地址是 contact@lpalmieri.com。

在本章以及之后的内容中，我建议你保持在浏览器中打开这几个网页：
[actix-web 的网站](https://actix.rs/)、[actix-web 的文档](https://docs.rs/actix-web/4.0.1/actix_web/index.html)和 [actix-web 的示例集合](https://github.com/actix/examples)。

## 3.3 我们的第一个端点：一个基本的健康检查

让我们从尝试实现一个健康检查端点开始：当我们收到一个 `/health_check` 端点的 GET 请求，我们将返回一个没有正文的 200 OK 响应。

我们可以使用 `/health_check` 端点来验证应用程序是否在线并且已准备好处理传入的请求。

将其与 pingdom.com 之类的 SaaS 服务结合使用，你可以在你的 API 异常时收到告警信息——对于你正在“边上”运行的邮件列表服务来说，这是一个很好的基线。

如果你正在使用某个容器编排系统（例如 Kubernetes 或 Nomad）来处理你的应用程序，健康检查端点也是十分有用的：容器编排系统可以通过调用 `health-check` 来检测出 API 失去响应的情况并触发重新启动。

### 3.3.1 上手 actix-web

我们将从使用 actix-web 构建一个 “Hello World!” 程序开始。

```rust
use actix_web::{web, App, HttpRequest, HttpServer, Responder};

async fn greet(req: HttpRequest) -> impl Responder {
    let name = req.match_info().get("name").unwrap_or("World");
    format!("Hello {}!", &amp;name)
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .route("/", web::get().to(greet))
            .route("/{name}", web::get().to(greet))
    })
    .bind("127.0.0.1:8000")?
    .run()
    .await
}
```

让我们将以上代码粘贴到 `main.rs` 文件中。

快速运行`cargo check`（注脚 1：在我们的开发过程中，我们并不总是对生成可运行的二进制文件感兴趣：我们通常只想要知道我们的代码是否能够编译。 cargo check 正是为了服务这个用例而诞生的：它运行的检查与 cargo build 相同，但它不需要执行任何机器代码的生成。 因此它的速度要快得多，并且能够为我们提供更紧密的反馈循环。 有关更多详细信息，请参见[链接](https://doc.rust-lang.org/edition-guide/rust-2018/cargo-and-crates-io/cargo-check-for-faster-checking.html)。）：
```plain
error[E0432]: unresolved import `actix_web`
 --> src/main.rs:1:5
  |
1 | use actix_web::{web, App, HttpRequest, HttpServer, Responder};
  |     ^^^^^^^^^ use of undeclared crate or module `actix_web`

error[E0433]: failed to resolve: use of undeclared crate or module `tokio`
 --> src/main.rs:8:3
  |
8 | #[tokio::main]
  |   ^^^^^ use of undeclared crate or module `tokio`

error: aborting due to 2 previous errors
```

我们还未将 actix-web 和 tokio 添加到我们的依赖项列表中，因此编译器无法解析我们的导入。

我们可以手动修复这种情况，通过添加

```toml
#! Cargo.toml
# [...]

[dependencies]
actix-web = "4"
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
```

这些内容到我们的 `Cargo.toml` 的 `[dependencies]` 下面，或者我们也可以使用 `cargo add` 来快速添加 2 个 crate 的最新版本为项目的依赖项：
```bash
cargo add actix-web@4.0.0
```

`cargo add` 不是一个默认的 `cargo` 命令：它由 `cargo-edit` 提供，这是一个社区维护的 cargo 扩展（注脚 2：cargo 遵循与 Rust 标准库相同的理念：如果可能，新功能的增加由第三方 crates 进行探索，并在某个有意义的地方被上游引用（例如 [cargo-vendor](https://github.com/alexcrichton/cargo-vendor)））。你可以使用以下命令来安装它：

```bash
cargo install cargo-edit
```

再运行一遍`cargo check`，现在应该没问题了。

现在你可以使用 `cargo run` 启动应用程序，并做一次快速的手工测试：
```plain
curl http://127.0.0.1:8000
```
```plain
Hello World!
```

太酷了，成功跑起来了！

如果你愿意，可以用 `Ctrl+C` 来优雅地关闭此 web 服务器。

### 3.3.2 剖析一个 actix-web 应用

现在让我们回头仔细看看我们刚刚复制粘贴到 main.rs 文件中的内容。
```plain
//! src/main.rs
// [...]

#[tokio::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .route("/", web::get().to(greet))
            .route("/{name}", web::get().to(greet))
    })
    .bind("127.0.0.1:8000")?
    .run()
    .await
}
```

#### 3.3.2.1 Server - HttpServer

[HttpServer](https://docs.rs/actix-web/4.0.1/actix_web/struct.HttpServer.html) 是我们这个应用程序的支柱。它会处理以下事情：
- 应用程序应该在哪里监听传入的请求？ 一个 TCP 套接字（例如 127.0.0.1:8000）？ 还是一个 Unix 域套接字？
- 我们应该允许的最大并发连接数是多少？ 单位时间内处理多少新连接？
- 我们是否应该启用传输层安全性 (TLS)？
- 等等。

换句话说，HttpServer 处理所有传输级别的问题。

接着会发生什么？ 当和一个 API 客户端建立连接后，HttpServer 该如何做来处理客户端的请求？

#### 3.3.2.2 Application - App

App 是所有应用程序逻辑所在的地方：包括路由、中间件、请求处理程序等。

App 是一个组件，其工作是将传入的请求作为其输入并吐出响应。

让我们放大看看我们的代码片段：
```rust
App::new()
    .route("/", web::get().to(greet))
    .route("/{name}", web::get().to(greet))
```

App 其实是构建器模式的一个实际例子：new() 为我们提供了一个白板，接着可以使用流式 API 每次在其上添加一个新行为（即一个接一个的链式方法调用）。

在整本书的课程中，我们将根据需要了解 App 的大部分 API：在我们的旅程结束时，它的大部分方法你应该都至少接触过一次。
