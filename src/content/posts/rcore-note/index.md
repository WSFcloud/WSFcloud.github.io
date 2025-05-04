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

## tips

### git使用
```bash
git fetch
git checkout chx
git checkout --orphan new-chx
git add .
git commit -m "Initial chx commit"
git branch -D chx
git branch -m chx
git push -u origin chx --force
```


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
│   ├── console.rs # SBI控制台驱动程序，将打印字符的 SBI 接口进一步封装，用于实现文本输出
│   ├── entry.asm # 初始化设置
│   ├── lang_items.rs # 提供给Rust编译器的一些语义项
│   ├── linker-qemu.ld # 链接脚本
│   ├── logging.rs #提供日志功能
│   ├── main.rs # 主函数
│   └── sbi.rs # 调用RustSBI接口
└── target # 编译后的二进制文件
```

### 实践作业
彩色化LOG：ch1代码已实现，使用LOG=[TRACE/DEBUG/INFO/WARN/ERROR] make run即可控制日志输出级别。


## ch2

### 应用程序
约定寄存器a0~a6保存系统调用的参数，a0保存系统调用的返回值。寄存器a7用来传递syscall ID，这是因为所有的syscall都是通过ecall指令触发的，除了各输入参数之外我们还额外需要一个寄存器来保存要请求哪个系统调用。由于这超出了Rust语言的表达能力，我们需要在代码中使用内嵌汇编来完成参数/返回值绑定和ecall指令的插入。
```rust
// user/src/syscall.rs
use core::arch::asm;
fn syscall(id: usize, args: [usize; 3]) -> isize {
    let mut ret: isize;
    unsafe {
        asm!(
            "ecall",
            inlateout("x10") args[0] => ret,
            in("x11") args[1],
            in("x12") args[2],
            in("x17") id
        );
    }
    ret
}
```

### 代码分析
初始化AppManager的全局实例APP_MANAGER，找到link_app.S中提供的符号_num_app，并从这里开始解析出应用数量以及各个应用的起始地址。有些全局变量依赖于运行期间才能得到的数据作为初始值，这导致这些全局变量需要在运行时发生变化，即需要重新设置初始值之后才能使用。使用lazy_static!声明了一个AppManager结构的名为APP_MANAGER的全局实例，且只有在它第一次被使用到的时候，才会进行实际的初始化工作。
```rust
// os/src/batch.rs
lazy_static! {
    // 定义一个静态引用类型的APP_MANAGER变量，用于管理多个应用程序的信息
    static ref APP_MANAGER: UPSafeCell<AppManager> = unsafe {
        // 使用UPSafeCell包装AppManager实例，允许单线程下的可变访问
        UPSafeCell::new({
            // 声明外部C函数_num_app，用于获取应用程序的数量
            extern "C" {
                fn _num_app();
            }
            // 将_num_app函数的地址转换为指向usize类型的指针
            let num_app_ptr = _num_app as usize as *const usize;
            // 使用read_volatile从指针处读取应用程序数量，避免编译器优化
            let num_app = num_app_ptr.read_volatile();
            // 初始化app_start数组，用于存储每个应用程序的起始地址
            let mut app_start: [usize; MAX_APP_NUM + 1] = [0; MAX_APP_NUM + 1];
            // 从内存中获取应用程序起始地址的切片，长度为num_app + 1
            let app_start_raw: &[usize] =
                core::slice::from_raw_parts(num_app_ptr.add(1), num_app + 1);
            // 将app_start_raw切片的数据复制到app_start数组中
            app_start[..=num_app].copy_from_slice(app_start_raw);
            // 创建并返回AppManager实例，包含应用程序数量、当前应用程序索引和起始地址数组
            AppManager {
                num_app,
                current_app: 0,
                app_start,
            }
        })
    };
}
```

### Trap
在 Trap 触发的一瞬间， CPU 就会切换到 S 特权级并跳转到 stvec 所指示的位置。但是在正式进入 S 特权级的 Trap 处理之前，上面提到过我们必须保存原控制流的寄存器状态，这一般通过内核栈来保存。注意，我们需要用专门为操作系统准备的内核栈，而不是应用程序运行时用到的用户栈。使用两个不同的栈主要是为了安全性：如果两个控制流（即应用程序的控制流和内核的控制流）使用同一个栈，在返回之后应用程序就能读到 Trap 控制流的历史信息，比如内核一些函数的地址，这样会带来安全隐患。于是，我们要做的是，在批处理操作系统中添加一段汇编代码，实现从用户栈切换到内核栈，并在内核栈上保存应用程序控制流的寄存器状态。我们声明两个类型 KernelStack 和 UserStack 分别表示内核栈和用户栈，它们都只是字节数组的简单包装。
```rust
// os/src/batch.rs

const USER_STACK_SIZE: usize = 4096 * 2;
const KERNEL_STACK_SIZE: usize = 4096 * 2;

#[repr(align(4096))]
struct KernelStack {
    data: [u8; KERNEL_STACK_SIZE],
}

#[repr(align(4096))]
struct UserStack {
    data: [u8; USER_STACK_SIZE],
}

static KERNEL_STACK: KernelStack = KernelStack { data: [0; KERNEL_STACK_SIZE] };
static USER_STACK: UserStack = UserStack { data: [0; USER_STACK_SIZE] };
```

常数 USER_STACK_SIZE 和 KERNEL_STACK_SIZE 指出用户栈和内核栈的大小分别为 8KiB。两个类型是以全局变量的形式实例化在批处理操作系统的 .bss 段中的。我们为两个类型实现了 get_sp 方法来获取栈顶地址。由于在 RISC-V 中栈是向下增长的，我们只需返回包裹的数组的结尾地址，以用户栈类型 UserStack 为例：
```rust
// os/src/batch.rs
impl UserStack {
    fn get_sp(&self) -> usize {
        self.data.as_ptr() as usize + USER_STACK_SIZE
    }
}
```

Trap上下文（即数据结构 TrapContext ），类似前面提到的函数调用上下文，即在 Trap 发生时需要保存的物理资源内容。由于难以找到哪些寄存器无需保存，因此将所有寄存器全部保存。对于 CSR 而言，进入 Trap 的时候，硬件会立即覆盖掉 scause/stval/sstatus/sepc 的全部或是其中一部分。对于scause/stval，它总是在 Trap 处理的第一时间就被使用或者是在其他地方保存下来了，因此它没有被修改并造成不良影响的风险。而对于 sstatus/sepc 而言，它们会在 Trap 处理的全程有意义（在 Trap 控制流最后 sret 的时候还用到了它们），而且确实会出现 Trap 嵌套的情况使得它们的值被覆盖掉。所以需要将它们也一起保存下来，并在 sret 之前恢复原样。

Trap流程为：
- 应用程序通过 ecall 进入到内核状态时，操作系统保存被打断的应用程序的 Trap 上下文；

- 操作系统根据Trap相关的CSR寄存器内容，完成系统调用服务的分发与处理；

- 操作系统完成系统调用服务后，需要恢复被打断的应用程序的Trap 上下文，并通 sret 让应用程序继续执行。

在代码实现上，首先通过 __alltraps 将 Trap 上下文保存在内核栈上，然后跳转到使用 Rust 编写的 trap_handler 函数完成 Trap 分发及处理。当 trap_handler 返回之后，使用 __restore 从保存在内核栈上的 Trap 上下文恢复寄存器。最后通过一条 sret 指令回到应用程序执行。

### Trap管理
Trap 上下文保存和恢复的汇编代码。在批处理操作系统初始化的时候，我们需要修改 stvec 寄存器来指向正确的 Trap 处理入口点。
```rust
// os/src/trap/mod.rs

global_asm!(include_str!("trap.S"));

pub fn init() {
    extern "C" { fn __alltraps(); }
    unsafe {
        stvec::write(__alltraps as usize, TrapMode::Direct);
    }
}
```

保存 Trap 上下文的 __alltraps 的实现：
```asm
# os/src/trap/trap.S

.macro SAVE_GP n
    sd x\n, \n*8(sp)
.endm

.align 2
__alltraps:
    csrrw sp, sscratch, sp
    # now sp->kernel stack, sscratch->user stack
    # allocate a TrapContext on kernel stack
    addi sp, sp, -34*8
    # save general-purpose registers
    sd x1, 1*8(sp)
    # skip sp(x2), we will save it later
    sd x3, 3*8(sp)
    # skip tp(x4), application does not use it
    # save x5~x31
    .set n, 5
    .rept 27
        SAVE_GP %n
        .set n, n+1
    .endr
    # we can use t0/t1/t2 freely, because they were saved on kernel stack
    csrr t0, sstatus
    csrr t1, sepc
    sd t0, 32*8(sp)
    sd t1, 33*8(sp)
    # read user stack from sscratch and save it on the kernel stack
    csrr t2, sscratch
    sd t2, 2*8(sp)
    # set input argument of trap_handler(cx: &mut TrapContext)
    mv a0, sp
    call trap_handler
```

当 trap_handler 返回之后会从调用 trap_handler 的下一条指令开始执行，也就是从栈上的 Trap 上下文恢复的 __restore ：
```asm
# os/src/trap/trap.S

.macro LOAD_GP n
    ld x\n, \n*8(sp)
.endm

__restore:
    # case1: start running app by __restore
    # case2: back to U after handling trap
    mv sp, a0
    # now sp->kernel stack(after allocated), sscratch->user stack
    # restore sstatus/sepc
    ld t0, 32*8(sp)
    ld t1, 33*8(sp)
    ld t2, 2*8(sp)
    csrw sstatus, t0
    csrw sepc, t1
    csrw sscratch, t2
    # restore general-purpuse registers except sp/tp
    ld x1, 1*8(sp)
    ld x3, 3*8(sp)
    .set n, 5
    .rept 27
        LOAD_GP %n
        .set n, n+1
    .endr
    # release TrapContext on kernel stack
    addi sp, sp, 34*8
    # now sp->kernel stack, sscratch->user stack
    csrrw sp, sscratch, sp
    sret
```

Trap 在使用 Rust 实现的 trap_handler 函数中完成分发和处理：
```rust
// os/src/trap/mod.rs

#[no_mangle]
pub fn trap_handler(cx: &mut TrapContext) -> &mut TrapContext {
    let scause = scause::read();
    let stval = stval::read();
    match scause.cause() {
        Trap::Exception(Exception::UserEnvCall) => {
            cx.sepc += 4;
            cx.x[10] = syscall(cx.x[17], [cx.x[10], cx.x[11], cx.x[12]]) as usize;
        }
        Trap::Exception(Exception::StoreFault) |
        Trap::Exception(Exception::StorePageFault) => {
            println!("[kernel] PageFault in application, kernel killed it.");
            run_next_app();
        }
        Trap::Exception(Exception::IllegalInstruction) => {
            println!("[kernel] IllegalInstruction in application, kernel killed it.");
            run_next_app();
        }
        _ => {
            panic!("Unsupported trap {:?}, stval = {:#x}!", scause.cause(), stval);
        }
    }
    cx
}
```

### 系统调用功能
syscall 函数并不会实际处理系统调用，而只是根据 syscall ID 分发到具体的处理函数。

### 执行应用程序
调用 run_next_app 函数切换到下一个应用程序。此时 CPU 运行在 S 特权级，而它希望能够切换到 U 特权级。在 RISC-V 架构中，唯一一种能够使得 CPU 特权级下降的方法就是执行 Trap 返回的特权指令，如 sret 、mret 等。事实上，在从操作系统内核返回到运行应用程序之前，要完成如下这些工作：

- 构造应用程序开始执行所需的 Trap 上下文；

- 通过 __restore 函数，从刚构造的 Trap 上下文中，恢复应用程序执行的部分寄存器；

- 设置 sepc CSR的内容为应用程序入口点 0x80400000；

- 切换 scratch 和 sp 寄存器，设置 sp 指向应用程序用户栈；

- 执行 sret 从 S 特权级切换到 U 特权级。

它们可以通过复用 __restore 的代码来更容易的实现上述工作。我们只需要在内核栈上压入一个为启动应用程序而特殊构造的 Trap 上下文，再通过 __restore 函数，就能让这些寄存器到达启动应用程序所需要的上下文状态。


## ch3

### 多道程序加载和执行
应用的内存布局从 APP_BASE_ADDRESS 开始依次为每个应用预留一段空间。

执行应用程序需要设置应用程序返回的不同 Trap 上下文（Trap 上下文中保存了 放置程序起始地址的 epc 寄存器内容）：

- 跳转到应用程序（编号 $i$）的入口点 $entry$

- 将使用的栈切换到用户栈 $stack$

### 任务切换
操作系统的核心机制—— 任务切换 ，在内核中这种机制是在 __switch 函数中实现的。 任务切换支持的场景是：一个应用在运行途中便会主动或被动交出 CPU 的使用权，此时它只能暂停执行，等到内核重新给它分配处理器资源之后才能恢复并继续执行。把应用程序的一次执行过程（也是一段控制流）称为一个 任务 ，把应用执行过程中的一个时间片段上的执行片段或空闲片段称为 “ 计算任务片 ” 或“ 空闲任务片 ” 。当应用程序的所有任务片都完成后，应用程序的一次任务也就完成了。从一个程序的任务切换到另外一个程序的任务称为 任务切换 。

为了确保切换后的任务能够正确继续执行，操作系统需要支持让任务的执行“暂停”和“继续”。需要让程序执行的状态（也称上下文），即在执行过程中同步变化的资源（如寄存器、栈等）保持不变，或者变化在它的预期之内。不是所有的资源都需要被保存，事实上只有那些对于程序接下来的正确执行仍然有用，且在它被切换出去的时候有被覆盖风险的那些资源才有被保存的价值。这些需要保存与恢复的资源被称为 任务上下文 (Task Context) 。

与Trap控制流相同，它对应用是透明的，不同之处在于：

- 它不涉及特权级切换。

- 它的一部分是由编译器帮忙完成的。

事实上，任务切换是来自两个不同应用在内核中的 Trap 控制流之间的切换。当一个应用 Trap 到 S 模式的操作系统内核中进行进一步处理（即进入了操作系统的 Trap 控制流）的时候，其 Trap 控制流可以调用一个特殊的 __switch 函数。__switch 函数和一个普通的函数之间的核心差别是它会 换栈 。

当 Trap 控制流准备调用 __switch 函数使任务从运行状态进入暂停状态的时候，在准备调用 __switch 函数之前，内核栈上从栈底到栈顶分别是保存了应用执行状态的 Trap 上下文以及内核在对 Trap 处理的过程中留下的调用栈信息。由于之后还要恢复回来执行，我们必须保存 CPU 当前的某些寄存器，我们称它们为 任务上下文 (Task Context)。

Trap 控制流在调用 __switch 之前就需要明确知道即将切换到哪一条目前正处于暂停状态的 Trap 控制流，因此 __switch 有两个参数，第一个参数代表它自己，第二个参数则代表即将切换到的那条 Trap 控制流。

假设某次 __switch 调用要从 Trap 控制流 A 切换到 B，一共可以分为四个阶段，在每个阶段中我们都给出了 A 和 B 内核栈上的内容。

- 阶段 [1]：在 Trap 控制流 A 调用 __switch 之前，A 的内核栈上只有 Trap 上下文和 Trap 处理函数的调用栈信息，而 B 是之前被切换出去的；

- 阶段 [2]：A 在 A 任务上下文空间在里面保存 CPU 当前的寄存器快照；

- 阶段 [3]：这一步极为关键，读取 next_task_cx_ptr 指向的 B 任务上下文，根据 B 任务上下文保存的内容来恢复 ra 寄存器、s0~s11 寄存器以及 sp 寄存器。只有这一步做完后， __switch 才能做到一个函数跨两条控制流执行，即 通过换栈也就实现了控制流的切换 。

- 阶段 [4]：上一步寄存器恢复完成后，可以看到通过恢复 sp 寄存器换到了任务 B 的内核栈上，进而实现了控制流的切换。这就是为什么 __switch 能做到一个函数跨两条控制流执行。此后，当 CPU 执行 ret 汇编伪指令完成 __switch 函数返回后，任务 B 可以从调用 __switch 的位置继续向下执行。

从结果来看，我们看到 A 控制流 和 B 控制流的状态发生了互换， A 在保存任务上下文之后进入暂停状态，而 B 则恢复了上下文并在 CPU 上继续执行。