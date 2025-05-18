# 第一个练习

[course/stage3-1.pptx](https://github.com/LearningOS/2025s-arceos-plerks/blob/main/course/stage3-1.pptx)第28页：

`[print_with_color]: 支持带颜色的打印输出`

运行`make run A=exercises/print_with_color`

预期：字符串输出带颜色

# 实现
修改arceos/ulib/axstd/src/macros.rs中的一行即可。

```Rust
/// Prints to the standard output, with a newline.
#[macro_export]
macro_rules! println {
    () => { $crate::print!("\n") };
    ($($arg:tt)*) => {
        // $crate::io::__print_impl(format_args!("{}\n", format_args!($($arg)*)));
        $crate::io::__print_impl(format_args!("\u{1B}[32m{}\u{1B}[0m\n", format_args!($($arg)*)));
    }
}
```
这个打印颜色的具体规则可以看第二阶段的[笔记](https://github.com/plerks/rcore-notes/blob/main/ch1/ch1.md#%E5%9B%9E%E5%88%B0rust_main)。