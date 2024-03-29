CPU Verification
================

.. contents:: Table of Contents


Importance
------------

CPU 作为中央处理器，在SoC中的重要性是显而易见的。
虽然对于学术流片来说，CPU 大概率只是作为一个控制器来使用，但在工程实现中，我们不能忽视 CPU 的验证。
理想情况下，我们编写的程序会运行在 CPU 上，通过 CPU 去控制加速器等模块，因此我们与加速器的交互是通过 CPU 来完成的，必须保证 CPU 功能的正确性。

相比于加速器来说，CPU 的设计比较复杂，我们不太可能从头自己写一个 CPU，大概率是 Github 上开源的 RISC-V CPU 或者 ARM Cortex-M0/M4。
M0/M4 有完整成熟的集成开发工具链，RISC-V CPU 相比而言生态相对比较简陋，各个部分需要我们自己拼装验证。
考虑到目前学术界流片的 SoC 所使用的 CPU 基本上都是 RISC-V CPU，因此接下来我们会补充一些关于 RISC-V CPU 的必要知识并简单介绍 RISC-V CPU 的验证流程。
以流片为目标，我们至少需要了解 CPU 基本的架构以及和外界交互的机制。



RISC-V Privileged Instruction Set Architecture
--------------

RISC-V Instruction Set
^^^^^^^^^^^^^^^^^

RISC-V指令集的一个特点是模块化，其核心是一个名为 RV32I 的基础 ISA，可运行完整的软件栈。
RV32I 已冻结，永不改变，这为编译器开发者、操作系统开发者和汇编语言程序员提供了稳定的指令目标。
模块化特性源于可选的标准扩展，硬件可根据应用程序的需求决定是否包含它们。
这种模块化特性能设计出面积小、能耗低的 RISC-V 处理器，这对于嵌入式应用至关重要。
RISC-V 编译器得知当前硬件包含哪些扩展后，便可为该硬件生成最优代码。
一般约定将扩展对应的字母加到指令集名称之后，以指示包含哪些扩展。
例如，RV32IMFDA 在必选基础指令集（RV32I）上添加了乘法（RV32M），单精度浮点（RV32F）, 双精度浮点（RV32D）和原子指令（RV32A）扩展。

.. note::

	RV32MFDA 以及 Zicsr，Zifencei 是 RISC-V 的标准扩展，它们与 RV32I 统称为 RV32G（G 代表 general），及 RV32G = RV32IMAFDZicsr_Zifencei。
	如果你注意过 ``RISC-V gcc`` 中的 ``-march`` 选项，你可能会发现它形如 ``rv32imac_zba_zbb_zbs_zbc_zicsr_zifencei``。
	这是因为，除了标准扩展之外，还有其他可选扩展，例如 ``zb*`` 指的是 “B” 标准扩展（Bit-manipulation）。

Privilege
^^^^^^^^^^^^^^^^^


`RISC-V 官方文档 <https://riscv.org/technical/specifications/>`__ 分为两大部分：非特权指令集（Unprivileged ISA）和特权指令集（Privileged ISA）。

Unprivileged ISA
#################

这部分定义了所有执行模式下都可以使用的基本指令集。
它包括整数、浮点、原子操作等基本指令，以及寄存器和基本的执行环境。
这些指令集构成了RISC-V程序的核心，是所有RISC-V兼容设备必须支持的基础。

Privileged ISA
#################

处理器的代码可以运行在不同等级的模式下，一般而言软件代码会默认运行在用户模式（*user mode*, U mode）下，该模式具有最低的权限。
除此以外，还有监管模式（*supervisor mode*, S mode）和机器模式（*machine mode*, M mode）。
操作系统一般运行在 S 模式下，而 M 模式则具有最高的特权，最重要的特性是拦截和处理异常（不寻常的运行时事件）。
在 M 模式下运行的代码能完全访问内存、I/O 和底层系统功能，这对启动和配置系统是必不可少的。
因此，M 模式是唯一一个所有标准 RISC-V 处理器都必须实现的特权模式。
应用程序（在U-mode下运行）通常无法直接执行特权指令，而是通过系统调用（Syscall）机制请求操作系统（在S-mode或M-mode下运行）提供服务，如访问硬件或管理资源。

.. note::

    实际上简单的 RISC-V 微控制器仅支持 M 模式。
	我们流片的 CPU 支持 U、S、M 模式，可以启动 Linux。


Control Status Register
^^^^^^^^^^^^^^^^^

体系结构的课程中一定会学习到寄存器堆（Register File），这些寄存器也被称为 GPR（General Purpose Register）。
实际上还有另一个“寄存器堆”——控制状态寄存器（CSR），它们被用来实现特权架构所带来的新特性，例如 :code:`mcause` 用于记录异常和中断的原因。
除了处理特权架构，还有一些 CSR 用于标识处理器特性或测量性能，例如 :code:`mcycle` 用于记录运行周期数。
CSR 记录了 CPU 当前的状态信息，因此对于仿真或者流片后验证都十分重要。

Assembly
------------------

了解了 CPU 的基本架构之后，我们需要知道软件代码如何翻译成 CPU 可运行的指令，这个过程被称为编译（compiling）。
将 C 程序翻译成计算机可运行的机器语言程序需要四个经典步骤：

:code:`foo.c` --compiler--> :code:`foo.s` --assembler--> :code:`foo.o` --linker--> :code:`a.out` --loader--> CPU

.. note::

    这些步骤是概念上的，实际上会合并某些步骤来加速翻译过程。

Compiler & Assembler
^^^^^^^^^^^^^^^

编译器负责将高级语言转换成汇编，汇编器负责将汇编转换成机器码。
汇编器的作用不仅是用处理器可理解的指令生成目标代码，还支持一些对汇编语言程序员或编译器开发者有用的操作。
这类操作是常规指令的巧妙特例，称为伪指令。
最经典的例子为 :code:`nop`，它在 RISC-V 中由 :code:`addi x0, x0, 0` 实现。

.. note::

	在程序员视角下，32个GPR有不同于 x0 ~ x31 的名称，这被称为 ABI （Application Binary Interface）。


下面是一段汇编

.. code-block::

	.text 			# 指示符：进入代码节
	.align 2 		# 指示符：将代码按 2^2 字节对齐
	.globl main 		# 指示符：声明全局符号 main
	main: 			# main 的开始符号
	addi sp,sp,-16 		# 分配栈帧
	sw ra,12(sp) 		# 保存返回地址
	lui a0,%hi(string1) 	# 计算 string1
	addi a0,a0,%lo(string1) # 的地址
	lui a1,%hi(string2) 	# 计算 string2
	addi a1,a1,%lo(string2) # 的地址
	call printf 		# 调用 printf 函数
	lw ra,12(sp) 		# 恢复返回地址
	addi sp,sp,16 		# 释放栈帧
	li a0,0 		# 装入返回值 0
	ret 			# 返回
	.section .rodata 	# 指示符：进入只读数据节
	.balign 4 		# 指示符：将数据按 4 字节对齐
	string1: 		# 第一个字符串符号
	.string "Hello, %s!\n" 	# 指示符：以空字符结尾的字符串
	string2: 		# 第二个字符串符号
	.string "world" 	# 指示符：以空字符结尾的字符串



以英文句号开头的命令称为汇编器指示符（assembler directives）。
这些命令作用于汇编器，而非由其翻译的代码，具体用于通知汇编器在何处放置代码和数据、指定程序中使用的代码和数据常量等。

.. note::

	汇编器生成的文件为 ELF（Executable and Linkable Format，可执行可链接格式）[TIS Committee 1995] 标准格式目标文件。

Linker
^^^^^^^^^^^^^^^

链接器允许分别编译和汇编各文件，故只改动一个文件时无需重新编译所有源代码。
链接器把新目标代码和已有机器语言模块（如函数库）“拼接” 起来，即编辑目标文件中所有 “跳转并链接（``jal``）” 指令的链接目标。
例如上述汇编有两个数据符号（``string1`` 和 ``string2``）和两个代码符号（``main`` 和 ``printf``）待确定。

根据链接的形式，可以将链接结果分为静态（static linking）和动态（dynamic linking）两种。
前者在程序运行前链接并加载所有库的代码，后者首次调用所需外部函数时才会将其加载并链接到程序中。

在编译和链接程序的过程中，通常会链接标准库和启动文件。
标准库（Standard Library）包含了许多常用的函数，例如输入输出函数、字符串处理函数等。
大多数程序都会使用到标准库中的函数，因此在链接阶段，编译器会将这些函数的代码链接到生成的可执行文件中。

启动文件（Start Files）是一些特殊的对象文件，它们包含了程序启动时需要执行的一些初始化代码。
例如，C 程序的入口点实际上是一个名为 start 或 _start 的函数，这个函数在启动文件中定义，它会设置好运行环境后再调用 main 函数。
具体的启动文件取决于你的编译器和操作系统。
例如，在使用 GCC 编译器的 Linux 系统中，启动文件通常是 ``crt1.o``、``crti.o``、``crtbegin.o``、``crtend.o`` 和 ``crtn.o``。
这些文件中的代码会设置堆栈，初始化全局变量，调用全局构造函数，等等。

.. note::

	当编译器选项中包含 ``-nostdlib`` 和 ``-nostartfiles`` 时，表示在链接阶段不链接标准库和启动文件。
	这通常在编写操作系统或嵌入式系统的代码时使用，因为这些系统可能没有标准库，或者需要自定义启动过程。
	需要注意的是，``-nostdlib`` 选项不仅会禁止链接 C 标准库，还会禁止链接启动文件和 GCC 的运行时库。
	如果你只想禁止链接 C 标准库，但仍然需要链接启动文件和 GCC 的运行时库，你可以使用 ``-nodefaultlibs`` 选项。

对象文件（.o 文件）是编译器生成的中间文件，它包含了源代码编译后的机器代码，但还没有被链接成可以执行的程序。
这些文件通常包含二进制数据，以及一些元数据，如符号表、重定位信息等。符号表中记录了源代码中的函数和变量的名称（符号）以及它们在机器代码中的位置。
重定位信息用于在链接阶段确定符号的最终地址。

.. hint::

	你可以使用一些工具来查看对象文件的内容。
	例如，你可以使用 ``objdump`` 工具来反汇编对象文件，查看它的汇编代码。你也可以使用 nm 工具来查看对象文件中的符号表。
	查看反汇编代码： ``objdump -d foo.o``；
	查看符号表： ``objdump -t your_file.o``；
	查看重定位信息：``objdump -r your_file.o``。


Loader
^^^^^^^^^^^^^^

运行一个程序时，加载器会将其加载到内存中，并跳转到它的起始地址。
可执行文件可以接收命令行参数。这些参数在程序启动时通过 main 函数的参数传递给程序。
main 函数的原型为 ``int main(argc, *argv[])``。

其中，``argc`` 是命令行参数的数量，``argv`` 是一个指向字符指针数组的指针，该数组包含了所有的命令行参数。
``argv[0]`` 是程序的名称，``argv[1]`` 是第一个命令行参数，以此类推。
最后一个元素 ``argv[argc]`` 是一个空指针。

例如，如果你的程序名为 ``prog``，并且你通过以下方式启动它：``./prog arg1 arg2``，那么 ``argc`` 将为 3，``argv[0]`` 将为 ./prog，``argv[1]`` 将为 arg1，``argv[2]`` 将为 arg2。

.. note::

	如今的 “加载器” 就是操作系统。

.. note::

	在进行交叉编译时，你的主机上的库（包括 C 标准库）通常不能直接用于目标系统。
	这是因为主机和目标系统可能有不同的架构（例如，主机可能是 x86，而目标系统是 RISC-V），并且它们可能有不同的操作系统接口（例如，主机可能是 Linux，而目标系统是 bare-metal）。

	因此，当你在 bare-metal RISC-V 环境中编译程序时，你需要一个为 RISC-V 架构和 bare-metal 环境定制的 C 库。
	这个库应该包含适合你的目标环境的函数实现，包括 ``exit`` 函数。

	如果你的程序使用了 C 库中的 ``exit`` 函数，但你没有提供一个适合你的目标环境的 ``exit`` 函数实现，那么在链接阶段，链接器会报错，因为它找不到 ``exit`` 函数的定义。

.. Tip::

	你可以查询 `RISC-V Assembly Programmer's Manual <https://github.com/riscv-non-isa/riscv-asm-manual/blob/master/riscv-asm.md>`__ 来了解如何编写 RISC-V 汇编语言。


Verification
------------------

Open-Sourced Tools
^^^^^^^^^^^^^^^^^^^

Instruction Set Simulator
######################

`Spike <https://github.com/riscv-software-src/riscv-isa-sim>`__ 是一个开源的 RISC-V ISA 仿真器。
它通过软件来模拟 CPU 指令的行为，属于行为级的仿真，速度较快。
我们通常认为 ISS 运行的结果是正确的。

Spike 仿真器中实现了两个重要的组件 HTIF（Host-Target Interface）和 fesvr （Front-End Server）。
它们在 Spike 仿真环境中有重要的作用，也可以作为单独的部件使用在其他的仿真环境中（如 Verilator）。

- HTIF 是一种用于在宿主机（通常是一台运行仿真器的计算机）和目标机（被仿真的 RISC-V 处理器）之间进行通信的机制。在测试中，HTIF 通常用于从 RISC-V 测试程序传递信息到仿真环境（如 Spike）。例如，通过写入特定的内存地址（如 tohost 和 fromhost），测试程序可以向宿主机发送信号以指示测试结果或进行调试。
- fesvr 是一个运行在宿主机上的软件，它作为仿真环境的一部分，用于与 RISC-V 目标机进行交互。fesvr 提供了一系列功能，包括加载程序到目标机、执行 I/O 操作以及处理目标机的系统调用请求。


RTL Simulator
#####################

`Verilator <https://www.veripool.org/verilator>`__ 是一个开源的 Verilog/SystemVerilog 仿真器。
它将 RTL 编译为 C++ 或 SystemC 后再运行仿真。
Verilator 是一个基于周期的仿真器，这意味着它不会评估单个时钟周期内的时间，也不会模拟精确的电路时序。
相反，电路状态通常每个时钟周期评估一次，因此无法观察到任何周期内毛刺，并且不支持定时信号延迟。

当使用 Verilator 对 RISC-V CPU 进行仿真并执行二进制文件时，流程大致如下：

- fesvr 加载二进制文件到仿真的 CPU。
- 仿真过程开始，CPU 开始执行加载的程序。
- 程序运行过程中可能会有系统调用或 I/O 请求，这些通过 HTIF 传递给 fesvr 处理。
- 如果程序需要向外部环境报告状态（如测试结果），它会写入特定的 tohost 地址。
- Verilator 监视 tohost 地址，根据写入的值执行相应操作（例如，如果 tohost 指示测试结束，Verilator 可以结束仿真过程）。

.. note::

	Verilator 的 testbench 需要用 C++ 或 SystemC 编写。

Environment
##################

`RISCV-DV <https://github.com/chipsalliance/riscv-dv>`__ 是一个随机的指令生成器，它可以给待测试的模块提供验证环境。

``tohost`` 是一个常用于 RISC-V 测试的机制，它是一种特殊的内存映射寄存器或地址，用于与测试环境通信。
在进行 RISC-V 的仿真或实际硬件测试时，``tohost`` 用于从正在运行的测试程序向测试环境（比如仿真器或测试框架）发送消息。
这些消息通常包括测试结果、调试信息或控制命令。例如，当测试程序完成或遇到错误时，它会将特定的值写入 ``tohost`` 地址，测试环境监视这个地址，根据写入的值判断测试状态或执行相应的操作。

在实际的硬件实现中，``tohost`` 并不是必须的，也不是 RISC-V 指令集架构（ISA）的一部分。
真实的硬件系统通常不需要像 ``tohost`` 这样的仿真特定机制。
硬件上的通信和调试功能通常是通过其他方式实现的，例如使用 JTAG 接口、串行端口、或者其他定制的硬件调试工具。

``tohost`` 地址通常在以下几个地方设置：

- 仿真环境: 在仿真环境（如 Spike）中，``tohost`` 地址需要在仿真器的内存映射中明确指定。这样仿真器可以捕捉到写入这个地址的操作，并据此处理测试结果。
- 测试程序: 在编写测试程序时，``tohost`` 地址会被定义为一个全局变量或宏。测试程序通过向这个地址写入特定的值来与测试框架通信，比如表示测试通过或失败。

Methodology
^^^^^^^^^^^^^^^^

Differential Testing
##################

进行 DiffTest 需要提供一个和 DUT（Design Under Test，测试对象）功能相同但实现方式不同的 REF（Reference，参考实现），然后让它们接受相同的有定义的输入，观测它们的行为是否相同。
在 CPU 验证中 DUT 为 RTL 仿真的结果，REF 为 ISS 仿真的结果。

Regression Testing
################

为了保证加入的新功能没有影响到已有功能的实现, 还需要重新运行测试用例，这个过程称为回归测试。
RISC-V 有多种回归测试的用例：

- `RISC-V Compliance <https://github.com/lowRISC/riscv-compliance>`__

- `RISC-V Tests <https://github.com/riscv-software-src/riscv-tests>`__

- `RISC-V Architecure Tests <https://github.com/riscv-non-isa/riscv-arch-test>`__

.. note::

	通过测试并不意味着设计符合 RISC-V 架构。这些只是基本的测试，检查规范的重要方面，而不关注细节。

CVA6 Example
----------------

`CVA6 <https://github.com/openhwgroup/cva6>`__ 是一个经过流片验证的开源 RISC-V CPU。

请参考 `CVA6 验证指导文档 <https://edge-training-soc.readthedocs.io/zh-cn/latest/cpu.html>`__ 了解具体的仿真流程。