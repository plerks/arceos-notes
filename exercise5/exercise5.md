# 第五个练习

course/stage3-5_6.pptx第17页：

`[sys_map]: 为宏内核支持sys_mmap系统调用，实现对文件的映射`

执行
```shell
make payload
./update_disk.sh payload/mapfile_c/mapfile
make run A=exercises/sys_map/ BLK=y
```
测试。

运行的结果，打印出的东西里要有"Read back content: hello, arceos!"这一行内容才算成功。这个输出来自用户程序 arceos/payload/mapfile_c/mapfile.c，用户程序会在调 mmap() 映射`文件`内容(文件内容为"hello, arceos!")后再读一遍`内存`内容并打印出来。

如果执行过了`make clean`，这时`make payload`会提示没有 arceos/target/riscv64gc-unknown-none-elf/release/linker_riscv64-qemu-virt.lds。这时候要执行一下`make build`。

# 实现
主要要参考 arceos/tour/m_2_0/src/main.rs handle_page_fault() 和 arceos/tour/m_2_0/src/loader.rs 的代码。还有要找到用 arceos_posix_api::get_file_like() 这个函数实现从fd读文件。

# 支持 linux 应用
训练营视频讲解里提到了arceos想支持linux的原生应用。也就是说linux下能跑的程序，arceos上也能跑。

`make payload`应该是把linux应用程序给编译出来。`make payload -n`跟着看一下，能看到是用`riscv64-linux-musl-gcc`把payload编译出来，例如把 arceos/payload/mapfile_c/mapfile.c 编译出来。

`./update_disk.sh payload/mapfile_c/mapfile`应该是生成 disk_img 文件，会把 payload/mapfile_c/mapfile 写进去，disk_img 是文件系统镜像。这样文件系统里就有了编译好的linux应用。

## 为什么用`musl-gcc`编译而不用`gcc`？

训练营视频讲解里提到了，大概意思是`musl-gcc`编译出来的东西要轻量级一点，同样的源代码，`gcc`编译出来的要 os 支持更多的系统调用才能运行起来。

不妨来看下payload的源代码：

```Rust
// arceos/payload/mapfile_c/mapfile.c

void verify_file(const char *fname)
{
    int fd;
    int ret;
    char *addr = NULL;

    fd = open(fname, O_RDONLY);
    if (fd < 0) {
        printf("Open file error!\n");
        exit(-1);
    }
    addr = mmap(NULL, 32, PROT_READ, MAP_PRIVATE, fd, 0);
    if (addr == NULL || strcmp(addr, "hello, arceos!") != 0) { // 原本只有 if (addr == NULL)，加了个mmap后的内容比较
        printf("Map file error!\n");
        exit(-1);
    }
    printf("Read back content: %s\n", addr);
    close(fd);
}
```

payload 是用`musl-gcc`编译出来的，但是 arceos 也能跑，是因为 arceos 支持了必要的系统调用，这样`musl-gcc`编译出来的代码像 printf 才能正常work。

`make run A=exercises/sys_map/ BLK=y`，打印出来的日志里有系统调用的记录，例如 "handle_syscall [66] ..."，而在 arceos/exercises/sys_map/src/syscall.rs 中：`const SYS_WRITEV: usize = 66;`。所以说，arceos 配合实现了相关系统调用，这样`musl-gcc`编译出来的程序才能跑在 arceos 上。

# 当前执行流程
执行`make run A=exercises/sys_map/ BLK=y`，exercises/sys_map/ 的 main() 仍然是被 rust_main() 调用的，其是与os在一起的。只是 exercises/sys_map/ 的 main() 的作用是去加载执行文件系统中的 /sbin/mapfile，mapfile 是个用户程序，这样就实现了支持用户程序的执行。

如果 exercises/sys_map/ 本身是个shell，那样就是启动后 shell 等待用户输入操作。