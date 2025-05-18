# 第三个练习

course/stage3-2.pptx第35页：

`[alt_alloc]: 为内存分配器实现新的内存算法bump`

只能修改modules/bump_allocator组件的实现，支持bump内存分配算法。不能改其它部分。

执行`make run A=exercises/alt_alloc/`测试

# 实现
把arceos/modules/bump_allocator/src/lib.rs中的EarlyAllocator类实现即可。

这里EarlyAllocator的实现极度简化

首先，EarlyAllocator可以分配字节，也可以分配页，并共用同一块空间来进行分配。

对于字节分配，不去记录已分配的字节区域的开始位置及长度，仅仅记录一个分配次数count，alloc时count + 1，dealloc时count - 1。

当count为0时才回收所有ByteAllocator的区域(b_pos收到start那里)

**ByteAllocator**:

1. 不检查dealloc时给出的参数pos位置是否是已经分配的位置(信任调用者)
    bump_allocator的Cargo.toml引用了allocator依赖(<https://github.com/arceos-org/allocator.git>)，打开链接后，其Cargo.toml中有documentation地址，顺着找到BuddyByteAllocator的dealloc()实现：<https://arceos.org/allocator/src/allocator/buddy.rs.html#43-45>，也没检查pos合法性什么的。

2. 中间的一个位置如果alloc后dealloc了，然后又来一个alloc，就算能用中间那个空洞，也不去用，而是继续增长b_pos，只有count减为0，才能回收掉

**PageAllocator**:

按arceos/modules/bump_allocator/src/lib.rs上方的注释"For pages area, it will never be freed!"，不考虑页的回收

EarlyAllocator所掌管的内存区域是init()时注册给它的，不考虑调用add_memory()再增加可分配内存区域的情况：

这第三个练习`make run A=exercises/alt_alloc/`，是arceos/modules/alt_axalloc/src/lib.rs里的static GLOBAL_ALLOCATOR需要初始化一个EarlyAllocator用来实现内存分配，这才用到EarlyAllocator，但是GLOBAL_ALLOCATOR对外提供的add_memory()是unimplemented!()，所以这里EarlyAllocator.add_memory()也不用实现。

很多报错情况没管，比如内存不够，b_pos和p_pos相互越过的情况


这里bump_allocator正经做应该是可以用arceos/modules/bump_allocator/Cargo.toml的那个allocator依赖里的`BuddyByteAllocator`和`BitmapPageAllocator`组合出来的，依赖是写好在Cargo.toml里的，后面可以试试(TODO)。

# 梳理下练习3的编译结构

arceos的组件化程度很高，很多东西都定义为了crate (所以有很多个Cargo.toml)。这里分析下执行`make run A=exercises/alt_alloc/`时发生了什么，`make run A=exercises/alt_alloc/ -n`可以让make只打印要执行的命令而不实际执行。

（注意`make run A=exercises/alt_alloc/`只是把组件拿出来运行了，不是arceos的启动执行流程）

首先看 arceos/exercises/alt_alloc/Cargo.toml，其引入了 axstd 这个依赖：
```toml
[package]
name = "alt_alloc"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
axstd = { workspace = true, features = ["alt_alloc"], optional = true }
```
这里引入依赖时为什么是这种写法，workspace = true是什么意思，处理alt_alloc这个crate时是怎么知道axstd crate的路径在哪里的？

## 根Cargo.toml

arceos对多crate的管理，用到了Cargo的[Workspace](https://doc.rust-lang.org/book/ch14-03-cargo-workspaces.html)功能，Workspace的各字段定义见<https://doc.rust-lang.org/cargo/reference/workspaces.html>。arceos根目录的Cargo.toml中，定义了一个Workspace。arceos/Cargo.toml:

<details>

<summary>arceos/Cargo.toml</summary>

```toml
[workspace]
resolver = "2"

members = [
    "modules/axalloc",
    "modules/alt_axalloc",
    "modules/axconfig",
    "modules/axdisplay",
    ...
]

[workspace.package]
version = "0.1.0"
authors = ["Yuekai Jia <equation618@gmail.com>"]
license = "GPL-3.0-or-later OR Apache-2.0 OR MulanPSL-2.0"
homepage = "https://github.com/arceos-org/arceos"
documentation = "https://arceos-org.github.io/arceos"
repository = "https://github.com/arceos-org/arceos"
keywords = ["arceos", "kernel"]
categories = ["os", "no-std"]

[workspace.dependencies]
axstd = { path = "ulib/axstd" }
axlibc = { path = "ulib/axlibc" }

arceos_api = { path = "api/arceos_api" }
arceos_posix_api = { path = "api/arceos_posix_api", features = ["fs", "fd"] }
axfeat = { path = "api/axfeat" }

axalloc = { path = "modules/axalloc" }
alt_axalloc = { path = "modules/alt_axalloc" }
```
</details>

---

这里先解释下members、workspace.dependencies、workspace.package三个字段。

### members
[文档](https://doc.rust-lang.org/cargo/reference/workspaces.html#the-members-and-exclude-fields)，文档中提到了一点，在子目录下运行时，如果要用workspace，会自动往上级目录Cargo.toml中找workspace定义:

> When inside a subdirectory within the workspace, Cargo will automatically search the parent directories for a Cargo.toml file with a [workspace] definition to determine which workspace to use. The package.workspace manifest key can be used in member crates to point at a workspace’s root to override this automatic search. The manual setting can be useful if the member is not inside a subdirectory of the workspace root.

members字段中的路径决定了哪些crate属于Workspace，Cargo会将这些crate视为一个整体进行管理，当运行 `cargo build --all`或`cargo test --all`时，Cargo只会处理members中列出的crate。

### workspace.dependencies
[文档](https://doc.rust-lang.org/cargo/reference/workspaces.html#the-dependencies-table)

members与依赖解析无关，workspace的依赖必须通过`[workspace.dependencies]`声明。

通过`[workspace.dependencies]`声明的依赖，相当于注册到了workspace中，可以从workspace中引用，无需关注相对路径。例如 arceos/Cargo.toml 中定义了workspace中的一个 crate axstd ：
```toml
[workspace.dependencies]
axstd = { path = "ulib/axstd" }
```
则 arceos/exercises/alt_alloc/Cargo.toml 中可以直接这样引用 axstd crate:
```toml
[dependencies]
axstd = { workspace = true, features = ["alt_alloc"], optional = true }
```
alt_alloc无需知道axstd在哪里，只需当做在workspace根下进行引用即可。

试了下把 arceos/Cargo.toml 中 members 的`"ulib/axstd"`去掉，`make run A=exercises/alt_alloc/`仍可正常运行；把`[workspace.package]`中的`axstd = { path = "ulib/axstd" }`去掉，`make run A=exercises/alt_alloc/`编译报错。

### workspace.package
[文档](https://doc.rust-lang.org/cargo/reference/workspaces.html#the-package-table)

这个workspace.package用来定义一些可供子crate继承的元信息。例如在根Cargo.toml中：
```toml
# [PROJECT_DIR]/Cargo.toml
[workspace]
members = ["bar"]

[workspace.package]
version = "1.2.3"
authors = ["Nice Folks"]
description = "A short description of my package"
documentation = "https://example.com/bar"
```
则子crate可以用`{key}.workspace = true`的格式来使用：
```toml
# [PROJECT_DIR]/bar/Cargo.toml
[package]
name = "bar"
version.workspace = true
authors.workspace = true
description.workspace = true
documentation.workspace = true
```

总之，通过Workspace，把各个crate平坦地排在了一起。

## features条件编译
[features文档](https://doc.rust-lang.org/cargo/reference/features.html)

像上面 arceos/exercises/alt_alloc/Cargo.toml 中:
```toml
[dependencies]
axstd = { workspace = true, features = ["alt_alloc"], optional = true }
```
alt_alloc 除了依赖 axstd 这个crate，还控制打开了 axstd 的 "alt_alloc" feature。

在 axstd 的 Cargo.toml 中，通过`[features]`字段定义了 axstd 所有的各种feature，则 axstd 的代码中可以用这些feature来进行条件编译。例如exercise2.md中的例子：

```Rust
// arceos/ulib/axstd/src/lib.rs

#[cfg(feature = "alloc")]
pub mod collections;
```
表示 axstd 的 alloc feature 开启了的情况下，给 axstd 增加一个模块 collections。这样就达到了条件编译的效果。

### features的依赖传递
features可以有依赖关系，可以传递去控制子crate的行为。例如arceos/ulib/axstd/Cargo.toml的`[features]`中有`alt_alloc = ["arceos_api/alt_alloc", "axfeat/alt_alloc"]`，这表示如果 axstd 的 "alt_alloc" feature 如果开启，则 axstd 要依赖 arceos_api 这个crate的 "alt_alloc" feature（两段一个是crate名，一个是crate的feature名）。于是就控制了依赖的crate的行为。

`arceos_api/alt_alloc`，arceos_api表示crate名，alt_alloc表示arceos_api要开启的feature。那么：rust怎么知道axstd是否确实依赖arceos_api这个crate？arceos_api这个crate在哪里？

arceos/ulib/axstd/Cargo.toml的`[dependencies]`中写了：
```toml
# arceos/ulib/axstd/Cargo.toml

[dependencies]
axfeat = { workspace = true }
arceos_api = { workspace = true }
axio = "0.1"
axerrno = "0.1"
kspin = "0.1"
```
**crate之间的依赖关系还是通过`[dependencies]`定义的，features只是配置各个节点的特性。**

## 练习3的crate引用关系
回到最初的问题，`make run A=exercises/alt_alloc/`是怎么造成要修改`arceos/modules/bump_allocator/src/lib.rs`的？

make run 会去运行 arceos/exercises/alt_alloc。

arceos/exercises/alt_alloc 依赖 axstd crate 并且要打开 axstd 的 "alt_alloc" feature。

axstd 依赖 arceos_api ([dependencies]字段)，并且由于 axstd 使用 "alt_alloc" feature，则 arceos_api 要使用 "alt_alloc" feature。 (axstd的Cargo.toml, [features]字段, alt_alloc = ["arceos_api/alt_alloc", "axfeat/alt_alloc"])

arceos_api 依赖了 alt_axalloc。 (arceos/api/arceos_api/Cargo.toml中，alt_axalloc = { workspace = true, optional = true }，这个alt_axalloc，在根Cargo.toml中，path为"modules/alt_axalloc")

alt_axalloc 依赖 bump_allocator，这样才走到要修改`arceos/modules/bump_allocator/src/lib.rs`这个文件的。

### "alt_alloc" feature起了什么用？
先来思考这个问题，arceos_api同时依赖了axalloc和alt_axalloc，对应modules/axalloc和modules/alt_axalloc，这二者都有定义GLOBAL_ALLCATOR:
```Rust
#[cfg_attr(all(target_os = "none", not(test)), global_allocator)]
static GLOBAL_ALLOCATOR: GlobalAllocator = GlobalAllocator::new();
```
为什么没报重定义？

我猜是这样：

GLOBAL_ALLOCATOR其实是被axruntime的init_allocator()调用的，arceos/modules/axruntime/src/lib.rs中写了两个init_allocator()，分别由"alloc"和"alt_alloc" feature控制条件编译。

arceos_api 是依赖了 axruntime 的，arceos_api的"alt_alloc" feature 会通过 axfeat，最终让 axruntime 也使用 "alt_alloc" feature。这样axalloc和alt_axalloc这两个crate，最终有一个的函数是没有被用到的，于是最终不会被链接器链接进来，最终只有一个GlobalAllocator。