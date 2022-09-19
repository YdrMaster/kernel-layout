# 内核链接脚本

本文描述了一种设计模式，用于为内核或其他裸机应用程序项目提供安全的自定义链接脚本。下文介绍这种模式的实现方式，并在 [`/linker`](/linker) 目录中包含一个简单的示例。

## 原理

裸机应用程序常常缺乏一个加载器来构建其期望的内存布局，必须由应用程序自举地构造，这就需要应用程序拥有自省的能力，了解其链接过程和产物。因此，裸机应用程序几乎必然需要定制链接脚本并在链接时查找链接脚本中声明的静态符号。

以往，这个操作就是简单地在项目目录的某处放一个 `*.ld` 文件，写上一个链接脚本。然后在源码中任意地使用 `extern "C"` 加载符号。当然，试图加载不存在的符号会在链接时报错，所以这样做在某种程度上也是安全的。但一方面比较死板，一旦需要链接脚本具有某种可变性，比如基于 feature 或条件变量构造不同的布局，就必须使用各种奇怪的方式引入可变性。另外也不利于维护，对符号的使用分布在各处，一旦修改布局，必须全局搜索 `extern "C"` 来找到所有使用符号的位置。

Rust 使用 crate 的方式提供了解决此问题的独特方案。不止有 `/src` 里的项目代码可以依赖 crate，`build.rs` 文件描述的构建过程也能。也就是说，一个 crate 只要同时满足两个部分的要求（`no_std`）就能被两个部分同时依赖。可以利用这个特性在两个部分之间共享代码。

---

> 这本质上是因为构建过程 `build.rs` 和源码 `/src` 都使用 Rust。其他一些现代高级语言也有类似的特性，比如使用 Gradle/kts 构建的 kotlin 项目，尽管可能不如 Rust 这么自然。

---

## 操作

1. 建立项目

   建立一个 workspace 项目，使用 `cargo new --lib linker` 创建一个内部 crate，并把它添加到 workspace。

   添加一些代码，主要是链接脚本中的常量定义和与布局相关的结构体。

2. 定制链接脚本

   在需要定制链接脚本的 bin crate（以 [`/app`](/app) 为例）的 [`Cargo.toml`](/app/Cargo.toml) 中添加依赖关系：

   ```toml
   [build-dependencies]
   linker = { path = "../linker" }
   ```

   为项目创建 [`build.rs`](/app/build.rs) 使用 `linker` crate 提供的信息：

   ```rust
   fn main() {
       use std::{env, fs, path::PathBuf};

       let ld = PathBuf::from(env::var_os("OUT_DIR").unwrap()).join("linker.ld");
       fs::write(&ld, format!("START = {:#x};{}", linker::START, linker::BODY)).unwrap();

       println!("cargo:rerun-if-changed=build.rs");
       println!("cargo:rustc-link-arg=-T{}", ld.display());
   }
   ```

   以上代码在 target 目录中创建一个 `linker.ld` 文件，根据 linker crate 提供的信息构造链接脚本代码，然后指定编译时使用这个脚本文件。

3. 在应用程序代码中使用

   在 [`Cargo.toml`](/app/Cargo.toml) 中添加依赖关系：

   ```toml
   [dependencies]
   linker = { path = "../linker" }
   ```

   现在可以安全地使用链接脚本中的信息了！

   ```rust
   fn main() {
       println!("linker start: {:#x}", linker::START);
   }
   ```
