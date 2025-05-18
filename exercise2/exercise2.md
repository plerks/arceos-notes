# 第二个练习

course/stage3-1.pptx第38页：

`[support_hashmap]: 支持HashMap类型`

在axstd组件中，支持collections::HashMap

执行`make run A=exercises/support_hashmap`测试

# 实现
arceos/exercises/support_hashmap/src/main.rs 中：
```Rust
#[macro_use]
#[cfg(feature = "axstd")]
extern crate axstd as std;

use std::collections::HashMap;
```
用axstd crate伪装了std。所以去改 arceos/ulib/axstd，这个 axstd 是个 lib crate。

在 arceos/ulib/axstd/src/lib.rs 中增加:
```Rust
#[cfg(feature = "alloc")]
pub mod collections; // 我定义的模块
```
表示如果axstd的alloc feature是开启了的情况下(在arceos/exercises/support_hashmap/Cargo.toml中定义依赖axstd时，给axstd打开了这个feature)，给axstd增加一个模块(mod) collectons。（[feature](https://doc.rust-lang.org/cargo/reference/features.html)用来控制实现条件编译的效果）

现在可以用 axstd::collections 访问到我定义的这个模块。

arceos/ulib/axstd/src/collections结构:
```shell
arceos/ulib/axstd/src/collections
├── hashmap.rs
└── mod.rs
```

在 arceos/ulib/axstd/src/collections/mod.rs 中：
```Rust
pub mod hashmap; // 声明有个模块叫hashmap，对应hashmap.rs
pub use hashmap::HashMap; // 这样，我的collections模块就有个HashMap，可以用collections::HashMap访问到。（我的HashMap原本是collections::hashmap::HashMap，这行use将HashMap导出到collections模块根下(可以用最后一段名字访问到)，于是现在也可以通过collections::HashMap来访问它）
```
也即是将HashMap做了个[**Re-export**](https://doc.rust-lang.org/rustdoc/write-documentation/re-exports.html)。

然后在`arceos/ulib/axstd/src/collections/hashmap.rs`中写个简化的HashMap就行了。