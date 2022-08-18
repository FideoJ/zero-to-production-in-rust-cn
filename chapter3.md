# 第三章 注册新的订阅者

- [第三章 注册新的订阅者](#第三章-注册新的订阅者)
  - [3.1 我们的策略](#31-我们的策略)
  - [3.2 选择一个 Web 框架](#32-选择一个-web-框架)
  - [3.3 我们的第一个端点：一个基本的健康检查](#33-我们的第一个端点一个基本的健康检查)
    - [3.3.1 上手 actix-web](#331-上手-actix-web)
    - [3.3.2 剖析一个 actix-web 应用](#332-剖析一个-actix-web-应用)
      - [3.3.2.1 Server - HttpServer](#3321-server---httpserver)
      - [3.3.2.2 Application - App](#3322-application---app)
      - [3.3.2.3 Endpoint - Route](#3323-endpoint---route)
      - [3.3.2.4 Runtime - tokio](#3324-runtime---tokio)
    - [3.3.3 实现健康检查的 handler](#333-实现健康检查的-handler)
  - [3.4 我们的第一个集成测试](#34-我们的第一个集成测试)
    - [3.4.1 如何测试端点](#341-如何测试端点)

上一章我们定义了我们将要打造的产品（一个邮件列表服务！），并将目光集中到一组精确的需求里。现在是撸起袖子开始干活的时候了。

本章将先尝试实现该用例：
> 作为博客的访问者，
> 我希望订阅邮件列表，
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

为了让旅途更轻松一些，我们将增加一块垫脚石：在处理 `/subscriptions` 端点之前，我们先实现一个称为 `/health_check` 的端点。这个端点没有任何业务逻辑，但却是一个熟悉 web 框架并了解它各个不同活动部件的好机会。

我们将全程依靠持续集成流水线执行检查——假如你还没有将它设置好，请快速查看第一章（或者直接获取一个[现成的模板](https://www.lpalmieri.com/posts/2020-06-06-zero-to-production-1-setup-toolchain-ides-ci/#5-2-ready-to-go-ci-pipelines)）。

## 3.2 选择一个 Web 框架

我们应该使用哪个 web 框架来编写我们的 Rust API 呢？

本节内容原本是要对比当今各个可用的 Rust web 框架的优缺点。但我最后发现这些内容太长了，将其嵌入到此处没有什么意义，因此我将其发表为一篇衍生文章：为了深入了解 actix-web、rocket、tide 和 warp，请查询《[Choosing a Rust web framework, 2020 edition](https://www.lpalmieri.com/posts/2020-07-04-choosing-a-rust-web-framework-2020-edition/)》。

简而言之，截至 2022 年 3 月，actix-web 应该是你编写用于生产用途的 Rust API 的首选 web 框架——在过去几年里它被广泛地使用，其背后有一个庞大而健康的社区，并且它还运行于 tokio 之上，因此也大大降低了开发者不得不处理各个不同的异步运行时之间的不兼容性/互操作性的可能性。

尽管如此，tide、rocket 和 warp 也有着巨大的潜力，在 2022 年晚些时候，我们最终可能做出与此前不同的决定——假如你在使用另一个 web 框架来跟随本书的内容，我将很高兴想要看看你的代码！请发送电子邮件给我，地址是 contact@lpalmieri.com。

在本章以及之后的内容中，我建议你保持在浏览器中打开这几个网页：[actix-web 的网站](https://actix.rs/)、[actix-web 的文档](https://docs.rs/actix-web/4.0.1/actix_web/index.html)和 [actix-web 的示例集合](https://github.com/actix/examples)。

## 3.3 我们的第一个端点：一个基本的健康检查

让我们从尝试实现一个健康检查端点开始：当我们收到一个 `/health_check` 端点的 GET 请求，我们将返回一个没有正文的 200 OK 响应。

我们可以使用 `/health_check` 端点来验证应用程序是否在线并且已准备好处理传入的请求。

将其与 pingdom.com 之类的 SaaS 服务结合使用，你可以在你的 API 异常时收到告警信息——对于你正在“边上”运行的邮件列表服务来说，它有了一个很好的基准检查。

如果你正在使用某个容器编排系统（例如 Kubernetes 或 Nomad）来处理你的应用程序，健康检查端点也是十分有用的：容器编排系统可以通过调用 `/health-check` 来检测出 API 失去响应的情况并触发重新启动。

### 3.3.1 上手 actix-web

我们将从使用 actix-web 构建一个 “Hello World!” 程序开始。

```rust
use actix_web::{web, App, HttpRequest, HttpServer, Responder};

async fn greet(req: HttpRequest) -> impl Responder {
    let name = req.match_info().get("name").unwrap_or("World");
    format!("Hello {}!", &amp;amp;amp;amp;name)
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

快速运行`cargo check`（注脚 1：在我们的开发过程中，我们并不总是对生成可运行的二进制文件感兴趣：我们通常只想要知道我们的代码是否能够编译。`cargo check`正是为了服务这个用例而诞生的：它运行的检查与`cargo build`相同，但它不需要执行任何机器代码的生成。 因此它的速度要快得多，并且能够为我们提供更紧密的反馈循环。有关更多详细信息，请参见[链接](https://doc.rust-lang.org/edition-guide/rust-2018/cargo-and-crates-io/cargo-check-for-faster-checking.html)。）：

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

我们可以手动修复这种情况，只需要添加以下内容到我们的 Cargo.toml 的 `[dependencies]` 下面。

```toml
#! Cargo.toml
# [...]

[dependencies]
actix-web = "4"
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
```

或者我们也可以使用 `cargo add` 来快速添加这两个 crate 的最新版本作为项目的依赖项：

```bash
cargo add actix-web@4.0.0
```

`cargo add` 不是一个默认的 `cargo` 命令：它由 `cargo-edit` 提供，这是一个社区维护的 cargo 扩展（注脚 2：cargo 遵循与 Rust 标准库相同的理念：如果可能的话，新功能的增加由第三方 crates 进行探索，并在某个有意义的时候合入上游（例如 [cargo-vendor](https://github.com/alexcrichton/cargo-vendor)））。你可以使用以下命令来安装它：

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

```rust
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

接着会发生什么呢？ 当和一个 API 客户端建立连接后，HttpServer 该如何做来处理客户端的请求？

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

在整本书的旅程中，我们将根据需要了解 App 的大部分 API：在我们的旅程结束时，它的大部分方法你应该都至少接触过一次。

#### 3.3.2.3 Endpoint - Route

我们该如何向我们的应用程序添加一个新的端点？

route 方法可能是最简单的方法——毕竟它是在 “Hello World!” 示例中使用的！

route 有两个参数：

- path，一个字符串，为了用于动态的路径，它可能是模板化的（例如“/{name}”）；
- route，它是 Route 结构的一个实例。

Route 结合了一个 handler 和一组 guards。

guards 指定了请求必须满足的条件，满足条件后，请求才能“match”并被传递给 handler。 从实现的角度来看，guards 是 Guard trait 的一个实现：`Guard::check`正是其魔力所在。

我们的代码片段里

```plain
.route("/", web::get().to(greet))
```

“/”将匹配基本路径后面没有任何路径段的所有请求——也即“http://localhost:8000/”。

`web::get()` 是 `Route::new().guard(guard::Get())` 的缩写，也就是当且仅当请求的 HTTP 方法是 GET 时，它才会被传递给 handler。

你可以开始想象当一个新的请求传入时会发生什么：App 遍历所有已注册的端点，直到找到匹配的端点（需要路径模板和 guards 都满足），并将请求对象传递给 handler。

这不是 100% 准确的，但它暂时是一个足够好的用于记忆的模型。

那么，handler 又是什么样的？ 它的函数签名是什么？

我们目前只有一个例子，greet：

```rust
async fn greet(req: HttpRequest) -> impl Responder {
    [...]
}
```

greet 是一个异步函数，它以 HttpRequest 作为输入并返回实现了 Responder trait 的某个实例（注脚 3：impl Responder 使用了 Rust 1.26 中引入的 impl Trait 语法——你可以在[此处](/tencent/static/images/upload_fail.png)找到更多详细信息。）。 如果某个类型可以转换为 HttpResponse，该类型即实现了 Responder trait——各种常见类型（例如 strings、status codes、bytes、HttpResponse 等）对 Responder trait 有着现成的实现，如果需要的话，我们自己也可以编写相关的实现。

是不是我们所有的 handler 都需要具有相同的 greet 函数签名？

不！ actix-web 引入了某些被禁止的 trait 黑魔法，允许了 handler 使用各种不同的函数签名（尤其是输入参数）。 我们很快就会提到这个。

#### 3.3.2.4 Runtime - tokio

我们刚才从整个 HttpServer 出发，详细讨论了 Route 。现在，让我们重新再看看整个 main 函数：

```rust
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

这里的 `#[tokio::main]` 的作用是什么？ 那么，让我们删除它，看看会发生什么！`cargo check` 马上尖叫着报错：

```plain
error[E0277]: `main` has invalid return type `impl std::future::Future`
 --> src/main.rs:8:20
  |
8 | async fn main() -> std::io::Result<()> {
  | ^^^^^^^^^^^^^^^^^^^
  | `main` can only return types that implement `std::process::Termination`
  |
  = help: consider using `()`, or a `Result`

error[E0752]: `main` function is not allowed to be `async`
 --> src/main.rs:8:1
  |
8 | async fn main() -> std::io::Result<()> {
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  | `main` function is not allowed to be `async`

error: aborting due to 2 previous errors
```

`HttpServer::run` 是一个异步方法，因此我们需要 `main` 也是异步的，但是作为二进制文件的入口点，`main` 不能是一个异步函数。 为什么会这样？

Rust 中的异步编程是建立在 Future trait 之上的：future 代表一个可能还不存在的值。 所有的 future 都有一个叫 poll 的公有方法，必须调用该方法才能使 future 取得进展，并最终决定其最终值。 你可以认为 Rust 的 future 是惰性的：除非 future 被轮询，否则无法保证它们会执行完成。 与其他语言采用的“推模型”相比，这通常被描述为“拉模型”（注脚 4：可以查看 async/await 的 [release 说明](https://blog.rust-lang.org/2019/11/07/Async-await-stable.html#zero-cost-futures)以获取更多详细信息。[withoutboats](https://github.com/withoutboats) 在 Rust LATAM 2019 上的[演讲](https://www.youtube.com/watch?v=skos4B5x7qE)是关于该主题的另一个很好的参考资料。 如果你更喜欢书籍而不是演讲，请查看 [Futures Explained in 200 Lines of Rust.](https://cfsamson.github.io/books-futures-explained/introduction.html)）。

根据 Rust 的设计，其标准库不包含异步运行时：你应该将一个异步运行时作为依赖项引入你的项目，即在 Cargo.toml 的 `[dependencies]` 下再添加一个 crate。这种方法非常通用：你可以自由实现自己的运行时，针对你的用例的特定要求进行优化（可以参考 [Fuchsia 项目](http://smallcultfollowing.com/babysteps/blog/2019/12/09/async-interview-2-cramertj/#async-interview-2-cramertj)或 [bastion](https://github.com/bastion-rs/bastion) 的 actor 框架）。

这解释了为什么 main 不能是异步函数：谁该负责对它调用 poll 呢？

没有特殊的配置语法能够告诉 Rust 编译器你的某个依赖项是异步运行时（正如我们如何定制 [allocators](https://doc.rust-lang.org/1.9.0/book/custom-allocators.html)），而且公平地说，甚至没有关于什么是运行时的标准化定义（例如，一个 Executor trait）。

因此，你应该在 main 函数之上启动异步运行时，然后使用它来驱动你的 future 到最终完成。

你现在可能已经猜到 `#[tokio:: main]`的目的是什么了，但猜测还不足以让我们满意：我们想直接看看。

该怎么做呢？

`tokio::main` 是一个过程宏，这是一个引入 [cargo expand](https://github.com/dtolnay/cargo-expand) 的好机会，对于 Rust 开发工作而言，它是我们的瑞士军刀的一个绝佳补充：

```bash
cargo install cargo-expand
```

Rust 宏在 token 级别起作用：它们接收符号流（例如，在我们的例子中，就是整个 main 函数）并输出一个新的符号流，然后将其传递给编译器。 换句话说，Rust 宏的主要目的是代码生成。

我们如何调试或检查特定的宏背后发生了什么？ 你可以直接检查它输出的 tokens！

这正是 cargo expand 的亮眼之处：它能够扩展代码中的所有宏，而不将输出传递给编译器，它允许你单步执行并了解发生了什么。

让我们使用 cargo expand 来揭秘 `#[tokio::main]`：

```bash
cargo expand
```

不幸的是，失败了：

```plain
error: the option `Z` is only accepted on the nightly compiler
error: could not compile `zero2prod`
```

我们正在使用 stable 版本的编译器来构建、测试和运行我们的代码。 然而，cargo-expand 依赖 nightly 版本的编译器来扩展我们的宏。

你可以这样行安装 nightly 编译器：

```bash
rustup toolchain install nightly --allow-downgrade
```

由 rustup 安装的软件包的某些组件可能会在最新的 nightly 版本中损坏或丢失：--allow-downgrade 告诉 rustup 去查找并安装所有需要的组件都可用的最新 nightly 版本。

你可以使用 rustup default 来更改 cargo 以及其它由 rustup 管理的工具所使用的默认工具链。 在我们的例子中，我们不想切换到 nightly - 我们只在 cargo expand 时需要它。

幸运的是，cargo 允许针对某个命令指定工具链：

```bash
# 只对此命令的调用使用 nightly 版本的工具链
cargo +nightly expand
```

```rust
/// [...]

fn main() -> std::io::Result<()> {
    let body = async move {
        HttpServer::new(|| {
            App::new()
                .route("/", web::get().to(greet))
                .route("/{name}", web::get().to(greet))
        })
        .bind("127.0.0.1:8000")?
        .run()
        .await
    };
    tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .build()
        .expect("Failed building the Runtime")
        .block_on(body)
}
```

我们终于可以看看宏展开后的代码了！

`#[tokio::main]` 展开后传递给 Rust 编译器的 main 函数确实是同步的，这就解释了为什么让它编译通过没有任何问题。

关键的地方是：

```rust
tokio::runtime::Builder::new_multi_thread()
    .enable_all()
    .build()
    .expect("Failed building the Runtime")
    .block_on(body)
```

我们启动了 tokio 的异步运行时，并使用它来驱动 HttpServer::run 返回的 future 完成。换句话说，`#[tokio::main]` 的工作是：给我们一个能够定义异步 main 函数的错觉，而在底层，它只不过是获取我们异步的 main 函数体并编写必要的样板代码来使其在 tokio 的运行时之上运行。

### 3.3.3 实现健康检查的 handler

我们已经仔细研究了 actix_web 的 “Hello World!” 示例中所有令人兴奋的部分：例如 HttpServer、App、route 和 tokio::main。

我们肯定已经足够了解如何修改示例以使我们的健康检查按照预期工作：当我们在 `/health_check` 收到 GET 请求时，返回一个没有正文的 200 OK 响应。

让我们再看看我们出发的地方：

```rust
//! src/main.rs
use actix_web::{web, App, HttpRequest, HttpServer, Responder};

async fn greet(req: HttpRequest) -> impl Responder {
    let name = req.match_info().get("name").unwrap_or("World");
    format!("Hello {}!", &amp;amp;amp;amp;name)
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

首先，我们需要一个请求处理程序。 模仿 greet 函数，我们可以从该签名开始：

```rust
async fn health_check(req: HttpRequest) -> impl Responder {
    todo!()
}
```

我们说过 Responder 只不过是一个做类型转换到 HttpResponse 的 trait。 直接返回一个 HttpResponse 的实例应该就可以工作！

查看 [HttpResponse 的文档](https://docs.rs/actix-web/4.0.1/actix_web/struct.HttpResponse.html)，我们可以使用 `HttpResponse::Ok` 来获得一个 [HttpResponseBuilder](https://docs.rs/actix-web/4.0.1/actix_web/struct.HttpResponseBuilder.html) ，其状态码为 200。HttpResponseBuilder 暴露了一个丰富的流式 API 来逐步构建一个 HttpResponse 响应，但我们在这里不需要它：我们可以通过在构建器上调用 [finish](https://docs.rs/actix-web/4.0.1/actix_web/struct.HttpResponseBuilder.html#method.finish) 方法来获得一个正文为空的 HttpResponse。

将这些内容粘合在一起：

```rust
async fn health_check(req: HttpRequest) -> impl Responder {
    HttpResponse::Ok().finish()
}
```

快速运行 `cargo check` 来确认我们的 handler 是否正确。 仔细看看 HttpResponseBuilder 会发现它也实现了 Responder——因此我们可以省略我们的 `finish` 调用，将我们的 handler 简化为：

```rust
async fn health_check(req: HttpRequest) -> impl Responder {
    HttpResponse::Ok()
}
```

下一步是 handler 的注册——我们需要通过 `route` 把它添加到我们的 App：

```rust
App::new()
    .route("/health_check", web::get().to(health_check))
```

让我们看看全景图：

```rust
//! src/main.rs

use actix_web::{web, App, HttpRequest, HttpResponse, HttpServer, Responder};

async fn health_check(req: HttpRequest) -> impl Responder {
    HttpResponse::Ok()
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| App::new().route("/health_check", web::get().to(health_check)))
        .bind("127.0.0.1:8000")?
        .run()
        .await
}
```

`cargo check` 运行顺利，尽管它会抛出一个警告：

```plain
warning: unused variable: `req`
 --> src/main.rs:3:23
  |
3 | async fn health_check(req: HttpRequest) -> impl Responder {
  |                       ^^^
  | help: if this is intentional, prefix it with an underscore: `_req`
  |
  = note: `#[warn(unused_variables)]` on by default
```

我们的健康检查的响应确实是静态的，并且不使用与传入的 HTTP 请求绑定的任何数据（路由除外）。 我们可以遵循编译器的建议并在 req 前加上下划线……或者我们可以完全从 `health_check` 函数中删除该输入参数：

```plain
async fn health_check() -> impl Responder {
    HttpResponse::Ok()
}
```

惊喜吧，它编译通过了！ actix-web 在幕后实现了一些非常高级的类型魔法，它能够接受范围广泛的各种签名的函数作为请求处理程序——之后会详细介绍。

剩下要做的事情是？

好吧，运行一个小测试！

```bash
# 首先，在另一个终端里使用 `cargo run` 启动应用程序
curl -v http://127.0.0.1:8000/health_check
```

```plain
* Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8000 (#0)
> GET /health_check HTTP/1.1
> Host: localhost:8000
> User-Agent: curl/7.61.0
> Accept: */*
>
< HTTP/1.1 200 OK
< content-length: 0
< date: Wed, 05 Aug 2020 22:11:52 GMT
```

恭喜，你刚刚实现了你的第一个能够工作的 actix_web 端点！

## 3.4 我们的第一个集成测试

`/health_check` 是我们的第一个端点，我们启动应用程序并使用 curl 执行手动测试，以验证一切都按照预期中工作。

然而手动测试是很费时间的：随着我们的应用程序变得越来越大，每当我们应用一些变更时，要验证它的行为是否仍然满足我们所有的假设，手动测试会变得越来越昂贵。

我们希望尽可能地自动化：每次我们提交变更时，这些检查都应该在我们的 CI 流水线中运行，以防止提交的变更对现有功能产生不利的影响。

即便健康检查的行为在我们的旅程中可能不会有太多的变更，这仍然是一个很好的起点，借此我们可以正确地设置好测试脚手架。

### 3.4.1 如何测试端点

API 其实是为了达到某一目的的一种手段：它是一个暴露于外部世界并执行某种任务（例如存储文档、发送电子邮件等）的工具。

我们在公开的 API 端点中定义了我们与 API 客户端之间的契约：关于系统输入和系统输出的某种共识协议，也即接口。

这个合约可能会随着时间的推移而发展，我们可以大致地描述两种情况：——向后兼容的变更（例如添加一个新的端点）；——不兼容的变更（例如，删除某个端点或者从其输出模式中删除某个字段）。

在第一种情况下，现有 API 客户端将继续按原样工作。在第二种情况下，如果现有的集成依赖于契约被违反的部分，则它们可能会无法工作。

即便我们可能会故意对我们的 API 契约引入不兼容的变更，但关键的是我们不要意外地破坏它。

如何检查我们是否造成了对用户可见的不利影响？最可靠的方法是什么？

通过与用户完全相同的方式与 API 交互，以此来测试 API：对它发送 HTTP 请求，对于所收到的响应，验证我们针对其的某些假设是否成立。

这通常被称为黑盒测试：我们检查系统在给定一组输入的情况下的输出，从而来验证系统的行为，并且这建立在无需访问其内部实现细节的前提下。

遵循这一原则，我们不会满足于直接调用 handler 函数的测试——举个例子：

```rust
#[cfg(test)]
mod tests {
    use crate::health_check;
    #[tokio::test]
    async fn health_check_succeeds() {
        let response = health_check().await;
        // 为了编译通过，你需要将`health_check`的返回值类型由`impl Responder`改为 `HttpResponse`
        // 你还需要使用`use actix_web::HttpResponse`完成导入！
        assert!(response.status().is_success())
    }
}
```

- 我们尚未检查 handler 是否在 GET 请求上被调用。
- 我们尚未检查 handler 是否以 `/health_check` 路径被调用。

更改这两个属性（译者注：HTTP 方法、HTTP 路径）中的任何一个都会破坏我们的 API 契约，但我们的测试仍然会通过——这不够好。

actix-web 提供了[某些便利](https://actix.rs/docs/testing/)，用于在不跳过路由逻辑的情况下与应用程序交互，但其方法存在严重的缺点：

- 迁移到另一个 Web 框架将迫使我们重写整个集成测试套件。我们希望集成测试与支撑 API 实现的技术尽可能地高度解耦（例如，当你进行大型重写或重构时，拥有与框架无关的集成测试简直是救命良药！）；
- 由于一些 actix-web 的限制，我们将无法在生产代码和测试代码之间共享我们的 App 启动逻辑，随着时间的推移，它们存在出现分歧的风险，建立在由我们的测试套件提供的某些保证之上的信任将会因此而被动摇。

我们将选择一个完全黑盒的解决方案：我们将在每次测试开始时启动我们的应用程序，并使用现成的 HTTP 客户端（例如 [reqwest](https://docs.rs/reqwest/0.11.0/reqwest/)）与之交互。
