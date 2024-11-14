---
title: rCore-OS 实验笔记
published: 2024-11-14
description: 'rCore-OS 实验笔记'
image: ''
tags: [笔记, Rust, OS]
category: '笔记'
draft: true
lang: 'zh_CN'
---

## ch1

:::note
#[panic_handler] 是一种编译指导属性，用于标记核心库core中的 panic! 宏要对接的函数（该函数实现对致命错误的具体处理）。该编译指导属性所标记的函数需要具有 fn(&PanicInfo) -> ! 函数签名，函数可通过 PanicInfo 数据结构获取致命错误的相关信息。这样Rust编译器就可以把核心库core中的 panic! 宏定义与 #[panic_handler] 指向的panic函数实现合并在一起，使得no_std程序具有类似std库的应对致命错误的功能。
:::

### qemu启动过程
qemu实际执行的第一条指令位于物理地址0x1000，接下来它将执行数条指令并跳转到被固化在qemu中的物理地址0x80000000对应的指令处并进入第二阶段，bootloader（rustsbi-qemu.bin——放在以物理地0x80000000开头的物理内存中），对计算机进行一些初始化工作，并跳转到下一阶段软件的入口，对于不同的bootloader而言，下一阶段软件的入口不一定相同，而且获取这一信息的方式和时间点也不同：入口地址可能是一个预先约定好的固定的值，也有可能是在bootloader运行期间才动态获取到的值。我们选用的RustSBI则是将下一阶段的入口地址预先约定为固定的0x80200000。在RustSBI的初始化工作完成之后，它会跳转到该地址并将计算机控制权移交给下一阶段的软件——也即我们的内核镜像os.bin。为了正确地和上一阶段的RustSBI对接，我们需要保证内核的第一条指令位于物理地址0x80200000处。为此，我们需要将内核镜像预先加载到qemu物理内存以地址0x80200000 开头的区域上。

通过链接脚本调整内核可执行文件的内存布局，使得内核被执行的第一条指令位于地址0x80200000处，同时代码段所在的地址应低于其他段。这是因为qemu物理内存中低于0x80200000的区域并未分配给内核，而是主要由RustSBI使用。其次，我们需要将内核可执行文件中的元数据丢掉得到内核镜像，此内核镜像仅包含实际会用到的代码和数据。

### 内核指令与函数调用
通过include_str!宏将同目录下的汇编代码entry.asm转化为字符串并通过global_asm!宏嵌入到代码中。
```rust
use core::arch::global_asm;
mod lang_items;
global_asm!(include_str!("entry.asm"));
```

__被调用者保存（Callee-Saved）寄存器__ ：被调用的函数可能会覆盖这些寄存器，需要被调用的函数来保存的寄存器，即由被调用的函数来保证在调用前后，这些寄存器保持不变；
__调用者保存（Caller-Saved）寄存器__：被调用的函数可能会覆盖这些寄存器，需要发起调用的函数来保存的寄存器，即由发起调用的函数来保证在调用前后，这些寄存器保持不变。

__sp__ 寄存器常用来保存栈指针（Stack Pointer），它指向内存中栈顶地址。在RISC-V架构中，栈是从高地址向低地址增长，在一个函数中，作为起始的开场代码负责分配一块新的栈空间，即将 __sp__ 的值减小相应的字节数即可，于是物理地址区间 $[新sp, 旧sp)$ 对应的物理内存的一部分便可以被这个函数用来进行函数调用上下文的保存/恢复，这块物理内存被称为这个函数的 __栈帧__ （Stack Frame）。同理，函数中的结尾代码负责将开场代码分配的栈帧回收，这也仅仅需要将 __sp__ 的值增加相同的字节数回到分配之前的状态。因此 __sp__ 是一个被调用者保存寄存器。我们需要保证它指向合法的物理内存，而且不能与程序的其他代码、数据段相交，因为在函数调用的过程中，栈区域里面的内容会被修改。

在第11行在内核的内存布局中预留了一块大小为 $4096 \times 16$ 字节也就是64KiB的空间用作接下来要运行的程序的栈空间。最开始的时候栈为空，栈顶和栈底位于相同的位置，我们用更高地址的符号 __boot_stack_top__ 来标识栈顶的位置。同时，我们用更低地址的符号 __boot_stack_lower_bound__ 来标识栈能够增长到的下限位置，它们都被设置为全局符号供其他目标文件使用。
```asm
# os/src/entry.asm
    .section .text.entry
    .globl _start
_start:
    la sp, boot_stack_top
    call rust_main

    .section .bss.stack
    .globl boot_stack_lower_bound
boot_stack_lower_bound:
    .space 4096 * 16
    .globl boot_stack_top
boot_stack_top:
```

第5行我们将栈指针sp设置为先前分配的启动栈栈顶地址，这样Rust代码在进行函数调用和返回的时候就可以正常在启动栈上分配和回收栈帧了。第6行我们通过伪指令call调用Rust编写的内核入口点rust_main将控制权转交给Rust代码，该入口点在main.rs中实现。
```rust
// os/src/main.rs
#[no_mangle]
pub fn rust_main() -> ! {
    loop {}
}
```

通过宏将rust_main标记为#[no_mangle]使得它在编译后的二进制文件中保留原始的函数名称以避免编译器对它的名字进行混淆，不然在链接的时候，entry.asm将找不到main.rs提供的外部符号rust_main从而导致链接失败。在rust_main函数的开场白中，我们将第一次在栈上分配栈帧并保存函数调用上下文，它也是内核运行全程中最底层的栈帧。

在内核初始化中，需要先完成对.bss段的清零。这是内核很重要的一部分初始化工作，在使用任何被分配到.bss段的全局变量之前我们需要确保.bss段已被清零。全局符号sbss和ebss由链接脚本linker.ld给出，并分别指出需要被清零的.bss段的起始和终止地址,接下来只需遍历该地址区间并逐字节进行清零即可。
```rust
// os/src/main.rs
#[no_mangle]
pub fn rust_main() -> ! {
    clear_bss();
    loop {}
}

fn clear_bss() {
    extern "C" {
        fn sbss();
        fn ebss();
    }
    (sbss as usize..ebss as usize).for_each(|a| {
        unsafe { (a as *mut u8).write_volatile(0) }
    });
}
```

### RustSBI
RustSBI这样运行在Machine特权级的软件被称为Supervisor Execution Environment(SEE)，即Supervisor执行环境。两层软件之间的接口被称为Supervisor Binary Interface(SBI)，即Supervisor二进制接口。rCore基于SBI spec v1.0.0版本，里面约定了SEE要向OS内核提供哪些功能。[sbi-rt](https://github.com/rustsbi/sbi-rt)封装了调用SBI服务的接口，可以直接使用。
```rust
// os/src/sbi.rs
pub fn console_putchar(c: usize) {
    #[allow(deprecated)]
    sbi_rt::legacy::console_putchar(c);
}
```

### 代码分析

/os目录下代码树：
```bash
.
├── .cargo
│   └── config.toml # Cargo配置文件
├── Cargo.lock # 锁定项目的依赖项版本
├── Cargo.toml # Rust项目的主配置文件
├── Makefile # 构建脚本
├── scripts
│   └── qemu-ver-check.sh # qemu版本检查
├── src
│   ├── console.rs # SBI控制台驱动程序，用于实现文本输出
│   ├── entry.asm # 初始化设置
│   ├── lang_items.rs # Rust宏实现
│   ├── linker-qemu.ld # 链接脚本
│   ├── logging.rs #提供日志功能
│   ├── main.rs # 主函数
│   └── sbi.rs # 调用RustSBI功能
└── target # 编译后的二进制文件
```