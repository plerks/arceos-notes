# 第六个练习

course/stage3-7.pptx第22页：

`[simple_hv]: 实现最简单的Hypervisor，响应VM_EXIT常见情况`

执行
```shell
make payload
./update_disk.sh payload/skernel2/skernel2
make run A=exercises/simple_hv/ BLK=y
```
测试。

运行的结果，打印的东西里要有：
```shell
a0 = 0x6688, a1 = 0x1234
Shutdown vm normally!
```
输出即算成功。

实际VM_EXIT退出的处理已经写好了，这里要处理的是arceos/payload/skernel2/src/main.rs中 csrr a1, mhartid 和 ld a0, 64(zero) 这两条指令的异常情况。前者特权级不够指令非法，后者内存区域未映射。

这个练习要我们直接跳过这两条指令，并手动替这两条指令完成功能，让a0和a1的值是 VmExit 时 assert 的那样。（有点面向用例编程，但是训练营视频讲解里的意思应该就是这样）

# 可能遇到的问题
如果用的qemu的版本是7.0.0，运行起来后可能会发生卡死的情况，这时arceos/qemu.log会变得巨大无比。需要[安装](https://learningos.cn/rCore-Camp-Guide-2025S/0setup-devel-env.html#qemu)高一点的qemu版本，比如qemu9。这个卡死的情况不是一定会发生，所以我就用的7.0.0也把这个练习做了。

如果执行过了`make clean`，运行可能会提示 linker_riscv64-qemu-virt.lds、disk.img、pflash.img 没有等等，执行下`make build`、`make disk_img`、`make pflash_img`去生成。

# Hypervisor
先把训练营的视频讲解看了。

Hypervisor是为了支持创建虚拟机的。

risc-v在硬件层面上就对Hypervisor有支持。risc-v有个叫 Hypervisor支持的东西（H扩展），开启H扩展之后，对于 S态 ，会进一步分为 HS态(Hypervisor Supervisor) 和 VS态 (Virtual Supervisor)。在VS态下的客户操作系统会认为自己运行在S态下，看到一个完整的S态环境（实际是虚拟出来的）。而HS态则管理虚拟机。

（可能是因为旧版的qemu对这个risc-v特性支持不够，上面的运行才会卡死。）

course/stage3-7.pptx第15页：ISA寄存器misa第7位代表Hypervisor扩展的启用/禁止。

H扩展还会抽象出整套虚拟的寄存器，比如vsstatus等，运行在VS态的虚拟机（Guest）仍然认为寄存器叫sstatus（Guest无感），而运行在HS态的Hypervisor则可以用vsstatus等虚拟寄存器来与Guest交流。

总之，通过H扩展，原本的S态被进一步分为了HS态和VS态。原本`内核(S态) <-> 用户程序(U态)`，现在可以多一层关系：`Hypervisor(HS态) <-> 虚拟机Guest(VS态)`。这样就支持了虚拟化，于是，**就像内核可以支持运行多个用户程序，Hypervisor也可以支持运行多个虚拟机**。

把chatgpt的解释也贴过来：
```
+---------------------------+
| Guest 用户态（U）         | ← VS-U
+---------------------------+
| Guest 内核（S）           | ← VS-S（也叫 VS 态）
+---------------------------+
| Hypervisor                | ← HS（运行在 S 态，但启用了 H 扩展）
+---------------------------+
| Machine 模式（M）         | ← Bootloader / Firmware (如 OpenSBI)
+---------------------------+
| 硬件                      |
+---------------------------+
```

当一个 Guest OS 执行了 ecall 系统调用时：

它以为自己是在 S 态；

实际硬件触发的是 VirtualSupervisorEnvCall；

HS 态的 Hypervisor 捕获异常，读取 vsepc 和 vsstatus 等，模拟系统调用；

调用结束后，恢复 vs* 状态，重新进入 Guest 执行下一条指令。

# 练习6执行流程
练习6用的payload是这样的：
```Rust
// arceos/payload/skernel2/src/main.rs

#[no_mangle]
unsafe extern "C" fn _start() -> ! {
    core::arch::asm!(
        "csrr a1, mhartid",
        "ld a0, 64(zero)",
        "li a7, 8",
        "ecall",
        options(noreturn)
    )
}
```
这会被当成一个Guest操作系统的代码，然后 rust_main() -> arceos/exercises/simple_hv/src/main.rs main()，这时 exercises/simple_hv 就相当于一个Hypervisor，其会去把 /sbin/skernel2 运行起来。

这样就实现了：启动Hypervisor，创建虚拟机运行。

# 实现
如前所述，csrr a1, mhartid 和 ld a0, 64(zero) 触发异常后，手动完成其功能，面向用例编程，具体见代码。

有一点就是，要手动跳过异常的指令，trap返回后执行下一条指令（这里的trap可能可以叫做vtrap？），需要调整sepc，否则回去之后执行的是同一条指令，会卡死。

执行`riscv64-linux-gnu-objdump -D ./target/riscv64gc-unknown-none-elf/release/skernel2`，可以看到这两条导致异常的指令长度都是4，因此把sepc + 4就能跳到下一条指令执行了：`ctx.guest_regs.sepc += 4`。

rcore里设置sepc的地方在 os/src/trap/mod.rs: `cx.sepc += 4;`。不过risc-v能有2字节长的命令，这里rcore是为了方便起见，默认指令长为4？

如果要处理这个问题的话，应该是要根据trap类型得到原本的指令长度？