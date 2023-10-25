# rust-based-os-comp2023-autumn-note
记录OS Training Camp 2023秋 学习实践过程



### 阶段一 rustlings

阶段一主要是通过rustlings来学习rust语言的基本用法，因为之前有一点rust基础且个人事情较忙，就没有详细记录过程。

### 阶段二 rcore

##### day 1 2023.10.23

主要阅读了前两章的内容，准备实验环境，总体比较好懂，但是需要对链接、汇编有基本的认识。

##### day 2 2023.10.24

阅读第三章并完成了实验，前3章涉及到比较多的汇编代码，所以比较考验细心，只能慢慢磨。总体难度也不大，主要是汇编细节多，需要时间熟悉。

##### day 3 2023.10.25

阅读第四章，只看文档或对照文档看相关代码会比较容易乱，之后从rust_main() 函数开始逐步阅读分析操作系统启动都**按什么顺序做了什么事**，就比较清晰了。

todo 有一处不明白是 MapArea.copy_data(...) 为什么一定要从数据段的第一页的第一个字节开始复制而不能指定位置开始复制，只是为了实现方便或一种约定？


### 知识点记录

#### riscv汇编

在大多数只与通用寄存器打交道的指令中， rs 表示 **源寄存器** (Source Register)， imm 表示 **立即数** (Immediate)，是一个常数，二者构成了指令的输入部分；而 rd 表示 **目标寄存器** (Destination Register)，它是指令的输出部分。



##### 寄存器

* **zero(x0)** 恒为零，函数调用不会对它产生影响
*  **ra(x1)**  callee-saved  被调用者函数可能也会调用函数，在调用之前就需要修改 `ra` 使得这次调用能正确返回. 每个函数都需要在开头保存 `ra` 到自己的栈帧中，并在结尾使用 `ret` 返回之前将其恢复
* **sp(x2)** callee-saved 栈指针 (Stack Pointer) 寄存器，它指向下一个将要被存储的栈顶位置
* **fp(s0)** 它既可作为s0临时寄存器，也可作为栈帧指针（Frame Pointer）寄存器，表示当前栈帧的起始位置，是一个被调用者保存寄存器。fp 指向的栈帧起始位置 和 sp 指向的栈帧的当前栈顶位置形成了所对应函数栈帧的空间范围。
* **gp(x3) tp(x4)** 在一个程序运行期间都不会变化，因此不必放在函数调用上下文中。
* **a0~a7(x10~x17)** caller-saved 传递参数，a0和a1还用来保存返回值
* **t0~t6(x5~x7, x28~x31)** caller-saved 作为临时寄存器使用，在被调函数中可以随意使用无需保存
* **s0~s11(x8~x9,x18~x27)** callee-saved 作为临时寄存器使用，被调函数保存后才能在被调函数中使用



##### 调用规范

在 RISC-V 架构中，栈是从高地址向低地址增长的。在一个函数中，作为起始的开场代码负责分配一块新的栈空间，即将 `sp` 的值减小相应的字节数即可，于是物理地址区间[新sp, 旧sp) 对应的物理内存的一部分便可以被这个函数用来进行函数调用上下文的保存/恢复，这块物理内存被称为这个函数的 **栈帧** (Stack Frame)。同理，函数中的结尾代码负责将开场代码分配的栈帧回收，这也仅仅需要将 `sp` 的值增加相同的字节数回到分配之前的状态。这也可以解释为什么 `sp` 是一个被调用者保存寄存器。

```
Father StackFrame
------  <- fp
ra
------
prev fp
------
Callee-saved
------
Local Variables
------  <- sp
```



在系统调用中，约定寄存器 `a0~a6` 保存系统调用的参数， `a0` 保存系统调用的返回值，`a7` 用来传递 syscall ID

##### 特权级机制

注意，OS不是直接操作硬件的，隔了一个SBI

```
App (User)
---ABI
OS (Supervisor)
---SBI
SEE(Supervisor Execution Environmen) (Machine)
```

**控制状态寄存器  (CSR, Control and Status Register)**

S 模式的 CSR 寄存器

* **sstatus** `SPP` 等字段给出 Trap 发生之前 CPU 处在哪个特权级（S/U）等信息
* **sepc** 当 Trap 是一个异常的时候，记录 Trap 发生之前执行的最后一条指令的地址
* **scause**  描述 Trap 的原因
* **stval**  给出 Trap 附加信息
* **stvec**  控制 Trap 处理代码的入口地址
* **sscratch** 保存用户/内核栈地址 

当CPU在U模式下执行完ecall后，硬件自动完成以下事情

* `sstatus` 的 `SPP` 字段会被修改为 CPU 当前的特权级（U/S）。
* `sepc` 会被修改为 Trap 处理完成后默认会执行的下一条指令的地址。
* `scause/stval` 分别会被修改成这次 Trap 的原因以及相关的附加信息。
* CPU 会跳转到 `stvec` 所设置的 Trap 处理入口地址，并将当前特权级设置为 S ，然后从Trap 处理入口地址处开始执行。

完成处理后在S模式执行```sret``` 返回

* CPU 会将当前的特权级按照 `sstatus` 的 `SPP` 字段设置为 U 或者 S
* CPU 会跳转到 `sepc` 寄存器指向的那条指令，然后继续执行。

##### 指令

```asm
# 跳转指令 
# rd <- pc + 4
# pc <- pc + imm
jal rd imm[20:1]

# 跳转指令
# rd <- pc + 4
# pc <- rs + imm
jalr rd, (imm[11:0])rs

# 跳转指令(伪)
# 被汇编器翻译为 jalr x0, 0(x1)
ret

# store
# Stores (dereferences) from register t0 into memory address (sp + 8).
sd t0, 8(sp)

# an assembler pesudo instruction that puts the address of SYMBOL into t0.
la t0, SYMBOL

# Adds value of t0 to the value -10 and stores the sum into a0.
addi a0, t0, -10

# 从 S 模式返回 U 模式：在 U 模式下执行会产生非法指令异常
sret

# 处理器在空闲时进入低功耗状态等待中断：在 U 模式下执行会产生非法指令异常
wfi

# 刷新 TLB 缓存：在 U 模式下执行会产生非法指令异常
sfence.vma

# 将csr的值读到通用寄存器rd中，将通用寄存器rs读到csr中，当rd和rs相同时，起到交换值的作用
# RISC-V 中读写 CSR 是原子指令，且常规的数据处理和访存类指令只能操作通用寄存器而不能操作 CSR
csrrw rd, csr, rs

# 将csr的值写入rd
csrr rd, csr

```



#### gdb常用命令

##### 断点

```asm
# 在内核入口处
b *0x80200000
```

##### 查看指令

```asm
# 从当前 PC 值的位置开始，在内存中反汇编 5 条指令
x/5i $pc
```

#### 地址空间

默认情况下 MMU 未被启用，此时直接访问物理地址。通过修改S特权级的satp CSR寄存器来启用分页。后续内容记录都以SV39为基础。

satp寄存器保存有页表根节点的物理地址，因此进程切换时需要刷新satp，然后使用*sfence.vma* 清空整个TLB

SV39虚拟地址39位，页大小4K bytes(12位)，[38 : 12] 为 VPN(virtual page number) 剩余[11 : 0]为页内偏移

物理地址只使用56位而不是64位，前44位是PPN，后12位是页内偏移

页表项(PTE, Page Table Entry)

[53 : 10] 共44位，是PPN， [7 : 0]是8个标志位

TLB, Translation Lookaside Buffer: 页表缓存







