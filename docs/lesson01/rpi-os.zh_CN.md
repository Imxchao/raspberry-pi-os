## 1.1: RPi OS 简介 , 无套 "Hello, World!"

我们将通过编写一个短小的，无套 "Hello, World" 应用来开启我们的操作系统开发之旅。我假定你已经具备[先决条件](../Prerequisites.md)，并且做好了一切准备。如果没有，现在就可以准备了。

在我们开始之前，我想要做一个简单命名约定。从README文档中你会发现整个教程被分为几节课程。每节课程由几个独立的文件组成，我将其称之为“章”（现在你正读第1课，第1.1章）。一章被进一步分成带有标题的“章节”。这个命名约定允许我引用资料的不同部分。

另外，我要你注意教程中有很多源码示例。我通常会先提供完整的代码块，并逐行解释。

### 项目结构

每节课程源码的结构都是相同的。你可以在[这里](https://github.com/Imxchao/raspberry-pi-os/tree/master/src/lesson01)找到本节课源码。我们简单介绍一下这个文件夹的主要组成部分：
1. **Makefile** 我们会使用[make](http://www.math.tau.ac.il/~danha/courses/software1/make-intro.html)来编译内核。通过Makefile配置`make`的执行指令——编译和链接源代码。
1. **build.sh or build.bat** 如果你想使用Docker编译内核，你将会用到这些文件。你不用在你的笔记本中安装make程序和编译工具链。
1. **src** 所有的代码都在这个文件夹里。
1. **include** 所有的头文件都放在这里。

### Makefile

现在，让我们凑近点看看项目中的Makefile。make的基本功能就是自动的检测需要重新编译的代码片段，并重新编译它们。如果你不熟悉make和Makefile，我推荐你读一下[这篇](http://opensourceforu.com/2012/06/gnu-make-in-detail-for-beginners/)文章。
第一课用到的Makefile在[这里](https://github.com/Imxchao/raspberry-pi-os/blob/master/src/lesson01/Makefile)。下面是完整的Makefile：
```
ARMGNU ?= aarch64-linux-gnu

COPS = -Wall -nostdlib -nostartfiles -ffreestanding -Iinclude -mgeneral-regs-only
ASMOPS = -Iinclude 

BUILD_DIR = build
SRC_DIR = src

all : kernel8.img

clean :
    rm -rf $(BUILD_DIR) *.img 

$(BUILD_DIR)/%_c.o: $(SRC_DIR)/%.c
    mkdir -p $(@D)
    $(ARMGNU)-gcc $(COPS) -MMD -c $< -o $@

$(BUILD_DIR)/%_s.o: $(SRC_DIR)/%.S
    $(ARMGNU)-gcc $(ASMOPS) -MMD -c $< -o $@

C_FILES = $(wildcard $(SRC_DIR)/*.c)
ASM_FILES = $(wildcard $(SRC_DIR)/*.S)
OBJ_FILES = $(C_FILES:$(SRC_DIR)/%.c=$(BUILD_DIR)/%_c.o)
OBJ_FILES += $(ASM_FILES:$(SRC_DIR)/%.S=$(BUILD_DIR)/%_s.o)

DEP_FILES = $(OBJ_FILES:%.o=%.d)
-include $(DEP_FILES)

kernel8.img: $(SRC_DIR)/linker.ld $(OBJ_FILES)
    $(ARMGNU)-ld -T $(SRC_DIR)/linker.ld -o $(BUILD_DIR)/kernel8.elf  $(OBJ_FILES)
    $(ARMGNU)-objcopy $(BUILD_DIR)/kernel8.elf -O binary kernel8.img
``` 
现在，让我们仔细的查看这个文件：
```
ARMGNU ?= aarch64-linux-gnu
```
Makefile 开始于一个变量定义。`ARMGNU`是交叉编译器的前缀。由于我们需要在`X86`机器上编译`arm64`架构的代码，所以需要用到[交叉编译器](https://en.wikipedia.org/wiki/Cross_compiler)。因此我们用`aarch64-linux-gnu-gcc`来替换`gcc`。

```
COPS = -Wall -nostdlib -nostartfiles -ffreestanding -Iinclude -mgeneral-regs-only
ASMOPS = -Iinclude 
```

`COPS` 和 `ASMOPS`是编译C代码和汇编代码时，依次传递给编译器的选项。这些选项需要作一个简单的说明：

* **-Wall** 显示所有警告信息。
* **-nostdlib** 不使用C标准库。C标准库中的大多数据调用最终都是和操作系统交互。我们写的是无套程序，没有任何底层操作系统，所以C标准库对我们来说没什么用处。
* **-nostartfiles** 不使用标准的启动文件。启动文件负责设置一个初始的堆栈指针，初始化静态数据，并跳至main入口点。这些我们都自己做。
* **-ffreestanding** 一个独立的环境，可能没有标准库，也可能没有程序启动所需的main函数. `-ffreestanding`选项指示编译器不要假定那些标准函数有其通常的定义。
* **-Iinclude** 在`include`文件夹中寻找头文件。
* **-mgeneral-regs-only** 只使用通用寄存器。 ARM处理器也有[NEON](https://developer.arm.com/technologies/neon)寄存器。我们不想用它，因为它增加了额外的复杂性（例如，我们会在切换上下文期间存储寄存器）。

```
BUILD_DIR = build
SRC_DIR = src
```

`SRC_DIR` 和 `BUILD_DIR` 目录依次包含源代码和编译过的目标文件。

```
all : kernel8.img

clean :
    rm -rf $(BUILD_DIR) *.img 
```

接下来，我们定义make目标。前两个目标非常简单：目标`all`是默认的那个，每当你键入`make`不带任何参数时，它都会被执行（`make`总是将第一个目标作为默认目标）此目标就是将所有的工作重定向到不同的目标，`kernel8.img` 。 
目标`clean`负责删除所有的编译产物和编译好的内核映像。

```
$(BUILD_DIR)/%_c.o: $(SRC_DIR)/%.c
    mkdir -p $(@D)
    $(ARMGNU)-gcc $(COPS) -MMD -c $< -o $@

$(BUILD_DIR)/%_s.o: $(SRC_DIR)/%.S
    $(ARMGNU)-gcc $(ASMOPS) -MMD -c $< -o $@
```

接下来的两个目标负责编译C和汇编文件。例如，如果在`src`目录中，我们有`foo.c`和`foo.S`，那么它们将被依次编译为`build/foo_c.o`和`build/foo_s.o`。`$<`和 `$@` 在make执行时被替换成输入和输出文件名（`foo.c`和 `foo_c.o`）。万一`build`目录不存在，我们在编译前就要创建它。

```
C_FILES = $(wildcard $(SRC_DIR)/*.c)
ASM_FILES = $(wildcard $(SRC_DIR)/*.S)
OBJ_FILES = $(C_FILES:$(SRC_DIR)/%.c=$(BUILD_DIR)/%_c.o)
OBJ_FILES += $(ASM_FILES:$(SRC_DIR)/%.S=$(BUILD_DIR)/%_s.o)
```

在这我们构建了所有目标文件的数组（`OBJ_FILES`），它是由C和汇编的源文件拼接而成。请看[替换引用](https://www.gnu.org/software/make/manual/html_node/Substitution-Refs.htm)。

```
DEP_FILES = $(OBJ_FILES:%.o=%.d)
-include $(DEP_FILES)
```

接下来的两行有点难搞。如果你看一看我们是怎样为C和汇编源文件定义编译目标的话，你会发现我们使用了`-MMD`参数。该参数指示`gcc`编译器给每个已生成的目标文件创建一个依赖文件。依赖文件定义了一个源文件的所有依赖。这些依赖通常是一个包含所有`header`的列表。我们需要包含所有已生成的依赖文件，这样当一个`header`改变时我们可以精准的知道哪一个需要重新编译。

```
$(ARMGNU)-ld -T $(SRC_DIR)/linker.ld -o kernel8.elf  $(OBJ_FILES)
``` 

我们使用`OBJ_FILES`数组构建`kernel8.elf`文件。我们使用链接器脚本`src/linker.ld`定义生成的可执行映像的基本布局（我们在下一个章节讨论链接器脚本）。

```
$(ARMGNU)-objcopy kernel8.elf -O binary kernel8.img
```

`kernel8.elf` 是[ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)格式的文件。问题是ELF文件是被设计成由操作系统执行的。为了写一个无套的程序，我们需要从ELF文件中，提取所有的可执行部分和数据部分，并将它们放到`kernel8.img`映像中去。后面的 `8` 表示 ARMv8，即 64 位架构。这个文件名称告诉硬件以64位模式启动处理器。
你也可以在`config.txt`文件中配置 `arm_control=0x200` 的标记来启动64位CPU。RPi OS 之前就是用的这个方式，你同样可以在一些练习题的答案中找到它。然而 `arm_control` 标记并没有文档化，最好是用`kernel8.img`的命名约定替换它。

### The linker script（链接器脚本）

链接器脚本主要是为了描述输入目标文件中哪些部分应该被映射到输出文件中（`.elf`）。更多关于链接器脚本的详情[请参见](https://sourceware.org/binutils/docs/ld/Scripts.html#Scripts)。现在，让我们看看 RPi OS 的链接器脚本：

```
SECTIONS
{
    .text.boot : { *(.text.boot) }
    .text :  { *(.text) }
    .rodata : { *(.rodata) }
    .data : { *(.data) }
    . = ALIGN(0x8);
    bss_begin = .;
    .bss : { *(.bss*) } 
    bss_end = .;
}
``` 

树莓派启动后，将`kernel8.img`加载到内存中，并从该文件的第一行开始执行。这就是为什么`.text.boot`部分必须放在第一行；我们将OS的启动代码放在这部分内。
`.text`, `.rodata`, 和 `.data` 部分包含内核编译指令，只读数据，和常规数据--这部分没有什么特别需要补充的。
`.bss`部分包含需要被初始置0的数据。通过将这样的数据单独放置，编译器能让ELF二进制文件节省一些空间--只有这部分的大小被存储在ELF的头部，而其本身的内容则被忽略了。当 image 被加载至内存后，我们必须将`.bss`部分置0；这就是为什么我们要记录这部分内容的开始和结束位置（因此才有`bss_begin` 和 `bss_end`符号），同时该部分需要对齐，因此它以8为倍数的地址`ALIGN(0x8)`开始。如果该部分不对齐，那么 `bss` 起始部分将难以用`str`指令存储0，因为`str`指令只能用于8字节对齐`8-byte-aligned`的地址。

### Booting the kernel（引导内核）

现在是时候看看 [boot.S](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson01/src/boot.S) 文件了。这个文件含有内核启动代码:

```
#include "mm.h"

.section ".text.boot"

.globl _start
_start:
    mrs    x0, mpidr_el1        
    and    x0, x0,#0xFF        // Check processor id
    cbz    x0, master        // Hang for all non-primary CPU
    b    proc_hang

proc_hang: 
    b proc_hang

master:
    adr    x0, bss_begin
    adr    x1, bss_end
    sub    x1, x1, x0
    bl     memzero

    mov    sp, #LOW_MEMORY
    bl    kernel_main
```
我们来仔细审查该文件:
```
.section ".text.boot"
```
首先，我们明确`boot.S`中的所有定义都要进入`.text.boot`部分。前面，我们看到这部分被链接器脚本放在了内核 image 的开头。所以内核启动时，从`start`函数开始执行：
```
.globl _start
_start:
    mrs    x0, mpidr_el1        
    and    x0, x0,#0xFF        // 检查处理器ID
    cbz    x0, master        // 所有非主CPU挂起
    b    proc_hang
```

这个函数首先要做的事就是检查处理器ID。树莓派 3 是四核处理器，设备通电后，每个核都开始执行相同的代码。然而，我们不想使用四个核；我们只想用第一个核，并且让其余的核陷入无限循环。这正是`_start`函数所负责的。它从系统寄存器[mpidr_el1](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.ddi0500g/BABHBJCI.html)中获取处理器ID。
如果当前处理器ID是0，则执行`master`函数：

```
master:
    adr    x0, bss_begin
    adr    x1, bss_end
    sub    x1, x1, x0
    bl     memzero
```

这里，我们调用`memzero`函数来清除`.bss`部分。我们稍候定义这个函数。在ARMv8架构中，为了方便，开始的7个参数通过x0-x6寄存器传入被调用函数。`memzero`函数只接收两个参数：起始地址（`bss_begin`）和被清除部分的大小（`bss_end - bss_begin`）。

```
    mov    sp, #LOW_MEMORY
    bl    kernel_main
```

当`.bss` 部分被清除后，我们初始化堆栈指针并将执行权传递给`kernel_main`函数。树莓派从地址 0 处加载内核；因此堆栈指针可以设置在任何足够高的地方，因为它并不会大到覆盖内核image的程度。`LOW_MEMORY`定义在[mm.h](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson01/include/mm.h)中，等于4MB。我们的内核的堆栈不会变的非常大，并且image本身也很小，所以4M对我们来说足够了。

对于那些不熟悉ARM的汇编语法的人，让我快速概述一下我们要用到的指令：

* [**mrs**](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289881374.htm) 将系统寄存器中的数值加载到（x0-x30）中的一个通用寄存器中。
* [**and**](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289863017.htm) 执行逻辑与操作。我们用这个命令删除从`mpidr_el1`寄存器中得到的值的最后一个字节。
* [**cbz**](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289867296.htm) 将先前执行的操作的结果和0比较，如果比较结果为真,就跳至（ARM中的术语叫`branch`，`分支`）其提供的标签处。
* [**b**](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289863797.htm) 对某个标签执行无条件分支。
* [**adr**](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289862147.htm) 加载标签的相对地址至目标寄存器中。本例中，我们需要指向`.bss`区域的开始和结束的指针。
* [**sub**](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289908389.htm) 两个寄存器的值相减。
* [**bl**](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289865686.htm) "带链接的分支": 执行一个无条件分支并将返回的地址存放到x30中（链接寄存器）。当子程序完成后，使用`ret`指令跳回至返回地址。
* [**mov**](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289878994.htm) 在寄存器之间移动值，或从常量移动至寄存器。

[这](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.den0024a/index.html) 是 ARMv8-A 开发者指南。如果你不熟悉ARM ISA的话，它将是个很好的资源。[这篇文章](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.den0024a/ch09s01s01.html) 特意列出了ABI中的寄存器使用约定。

### The `kernel_main` function（`kernel_main`函数）

我们已经看到引导代码最终将控制权交给了`kernel_main`函数。让我们瞧瞧：

```
#include "mini_uart.h"

void kernel_main(void)
{
    uart_init();
    uart_send_string("Hello, world!\r\n");

    while (1) {
        uart_send(uart_recv());
    }
}

```

这是内核中最简单的函数之一。它使用`Mini UART` 设备打印至屏幕并读取用户输入。内核仅仅打印了`Hello, world!`，然后就进入了无终止运行，不断的读取用户输入的字符并将其输出至屏幕。

### Raspberry Pi devices（树莓派中的设备）

现在我们将深入研究树莓派中的一些具体内容。在我们开始之前，我建议你下载[BCM2837 ARM 外设手册](https://github.com/raspberrypi/documentation/files/1888662/BCM2837-ARM-Peripherals.-.Revised.-.V2-1.pdf)。BCM2837是树莓派 3B和3B+使用的主板。在我们的讨论中，有时我也会提到BCM2835 和 BCM2836 - 这些是老版本的树莓派使用的主板。

在我们继续研究实现细节之前，我要分享一些使用内存映射（memory-mapped）设备的基本概念。BCM2837 是一个简单[SOC 芯片上的系统或单片系统](https://en.wikipedia.org/wiki/System_on_a_chip)的主板。在这样的一个主板上，通过内存映射寄存器访问所有设备。树莓派3 为设备保留了`0x3F000000`地址以上的内存。为了启用或配置特定设备，你需要向设备的一个寄存器中写入一些数据。一个设备寄存器只有32-bit的内存区域。`BCM2837 ARM 外设`手册中描述了每个设备寄存器中每一位的含义或用途。查看手册中第1.2.3节 ARM 物理设备地址和相关文档，了解有关我们为什么使用`0x3F000000`作为基地址的更多信息（尽管整个手册都使用`0x7E000000`）。

由 `kernel_main` 函数，你可能猜到我们将要使用 Mini UART 设备。UART 表示[通用异步收发传输器或串口](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter)。该设备能将一个内存映射寄存器中值转换成一个高低电平序列。该序列由`TTL-to-serial 线`传入你的电脑，被解释并显示到你电脑的终端命令行或仿真器中。我们将使用 Mini UART 来方便我们和树莓派的通信。如果你想了解 Mini UART 寄存器的规范，请查看`BCM2837 ARM 外设`手册的第8页。

树莓派有两个UARTs: Mini UART 和 PL011 UART。本课程中我们只用前一个，因为简单。然而，还有一个可选的，[exercise](https://github.com/Imxchao/raspberry-pi-os/blob/master/docs/lesson01/exercises.md)中介绍了如何使用PL011 UART。如果你想了解更多关于树莓派 UARTs 并学习它们的不同之处，你可以查看[官方文档](https://www.raspberrypi.org/documentation/configuration/uart.md)。

另一个需要的自己去熟悉的设备是GPIO[通用输入/输出](https://en.wikipedia.org/wiki/General-purpose_input/output)。GPIO负责控制`GPIO 引脚`。下面的图片中你应该轻松地识别他们：

![Raspberry Pi GPIO pins](../../images/gpio-pins.jpg)

GPIO 可以用来配置不同引脚的作用。例如，为了启用 Mini UART，我们需要激活引脚14和15并设置他们使用这个设备。下图展示了GPIO引脚对应的数字：

![Raspberry Pi GPIO pin numbers](../../images/gpio-numbers.png)

### Mini UART initialization （初始化 Mini UART）

现在，我们来看一看如何初始化 mini UART。这部分代码在[mini_uart.c](https://github.com/Imxchao/raspberry-pi-os/blob/master/src/lesson01/src/mini_uart.c)中：

```c
void uart_init ( void )
{
    unsigned int selector;

    selector = get32(GPFSEL1);
    selector &= ~(7<<12);                   // clean gpio14
    selector |= 2<<12;                      // set alt5 for gpio14
    selector &= ~(7<<15);                   // clean gpio15
    selector |= 2<<15;                      // set alt5 for gpio 15
    put32(GPFSEL1,selector);

    put32(GPPUD,0);
    delay(150);
    put32(GPPUDCLK0,(1<<14)|(1<<15));
    delay(150);
    put32(GPPUDCLK0,0);

    put32(AUX_ENABLES,1);                   //Enable mini uart (this also enables access to its registers)
    put32(AUX_MU_CNTL_REG,0);               //Disable auto flow control and disable receiver and transmitter (for now)
    put32(AUX_MU_IER_REG,0);                //Disable receive and transmit interrupts
    put32(AUX_MU_LCR_REG,3);                //Enable 8 bit mode
    put32(AUX_MU_MCR_REG,0);                //Set RTS line to be always high
    put32(AUX_MU_BAUD_REG,270);             //Set baud rate to 115200

    put32(AUX_MU_CNTL_REG,3);               //Finally, enable transmitter and receiver
}
``` 

这里我们用了两个函数 `put32` 和 `get32`。这些函数非常简单；它们允许我们从一个32-bit寄存器中读取并写入一些数据。你可以在[utils.S](https://github.com/Imxchao/raspberry-pi-os/blob/master/src/lesson01/src/utils.S)中看看它们是怎么实现的。`uart_init`是本课程中最复杂和最重要的函数之一，并且我们会在接下来的三个小节中继续探讨它们。 

#### GPIO alternative function selection （GPIO可选功能的选择）

首先，我们需要激活GPIO引脚。大多数引脚可以用于不同的设备，以至于在使用一个特定的引脚前，我们需要选择引脚的 `alternative function`，一个 `alternative function` 只是一个从0到5的数字，可以设置每个引脚并配置哪个设备可以连接它。你可以看到下图中是所有GIPO可选功能清单（图片取自`BCM2837 ARM 外设`手册的102页）：

![Raspberry Pi GPIO alternative functions](../../images/alt.png?raw=true)

你看引脚14和15的可选功能有 TXD1 和 RXD1 可供选择。这表示如果我们选择引脚14和15的可选功能数字5（`alt5`），那么它们会用作 Mini UART 的传输数据引脚和接收数据引脚。`GPFSEL1` 寄存器用于控制引脚10-19的可选功能。下表展示了这些寄存器中所有位的含义（`BCM2837 ARM 外设`手册的92页）。

![Raspberry Pi GPIO function selector](../../images/gpfsel1.png?raw=true)

所以，现在你明白了你需要理解下面几行代码的所需一切知识，用于配置GPIO引脚 14 和 15 使用 Mini UART 设备： 

```
    unsigned int selector;

    selector = get32(GPFSEL1);
    selector &= ~(7<<12);                   // clean gpio14
    selector |= 2<<12;                      // set alt5 for gpio14
    selector &= ~(7<<15);                   // clean gpio15
    selector |= 2<<15;                      // set alt5 for gpio 15
    put32(GPFSEL1,selector);
```

#### GPIO pull-up/down 

当你使用树莓派的GPIO时，你会经常遇到如pull-up/pull-down。关于这些概念[这里](https://grantwinney.com/using-pullup-and-pulldown-resistors-on-the-raspberry-pi/)有非常详细的解释。对于那些懒得完整地阅读文章的家伙，我会简单的解释下 pull-up/pull-down 的概念。

如果你用一个特定的引脚作为输入，但这个引脚什么都没有接，你将不能识别出这个引脚的值是1或0。事实上，设备会随机的上报数值。pull-up/pull-down 机制能让你克服这个问题。如果你设置引脚至pull-up状态并且没有任何连接，它将会始终报 `1` （pull-down 状态的这个值总是0）。在我们的案例中，我们不需要 pull-up 也不需要 pull-down 状态，因为14和15引脚会被一直连接。即使是重启后，引脚状态也会保留，因为在使用任何引脚之前，我们总是会初始化它的状态。这有三个状态可选：pull-up， pull-down， 和第三者（清除pull-up 或 pull-down state的状态），我们需要第三者。

切换引脚状态不是一个十分简单的过程，因为它需要切换一个物理上的电路板的开关。这个过程涉及了 `GPPUD` 和 `GPPUDCLK` 寄存器，`BCM2837 ARM 外设`手册的101页有描述这个过程。我复制了这部分描放这里：

```
The GPIO Pull-up/down Clock Registers control the actuation of internal pull-downs on
the respective GPIO pins. These registers must be used in conjunction with the GPPUD
register to effect GPIO Pull-up/down changes. The following sequence of events is
required:
1. Write to GPPUD to set the required control signal (i.e. Pull-up or Pull-Down or neither
to remove the current Pull-up/down)
2. Wait 150 cycles – this provides the required set-up time for the control signal
3. Write to GPPUDCLK0/1 to clock the control signal into the GPIO pads you wish to
modify – NOTE only the pads which receive a clock will be modified, all others will
retain their previous state.
4. Wait 150 cycles – this provides the required hold time for the control signal
5. Write to GPPUD to remove the control signal
6. Write to GPPUDCLK0/1 to remove the clock
``` 

这个过程描述了我们应该怎样清除一个引脚的 pull-up 和 pull-down 状态，这也是我们在下面的代码中对引脚14和15所做的过程：

```
    put32(GPPUD,0);
    delay(150);
    put32(GPPUDCLK0,(1<<14)|(1<<15));
    delay(150);
    put32(GPPUDCLK0,0);
```

#### Initializing the Mini UART（初始化 Mini UART）

现在我们的 Mini UART 已经连接到了 GPIO 引脚了，并且引脚也配置好了。`uart_init` 函数的剩余部分是专门初始化 Mini UART 的。

```
    put32(AUX_ENABLES,1);                   //Enable mini uart (this also enables access to its registers)
    put32(AUX_MU_CNTL_REG,0);               //Disable auto flow control and disable receiver and transmitter (for now)
    put32(AUX_MU_IER_REG,0);                //Disable receive and transmit interrupts
    put32(AUX_MU_LCR_REG,3);                //Enable 8 bit mode
    put32(AUX_MU_MCR_REG,0);                //Set RTS line to be always high
    put32(AUX_MU_BAUD_REG,270);             //Set baud rate to 115200

    put32(AUX_MU_CNTL_REG,3);               //Finally, enable transmitter and receiver
```
让我们一行一行的解释这个代码片段。

```
    put32(AUX_ENABLES,1);                   //Enable mini uart (this also enables access to its registers)
```
这一行的作用是来开启 Mini UART。我们必须首先做好，因为它同时也使我们能访问 Mini UART 其他所有的寄存器（即，除了AUX_ENABLES之外所有的寄存器）。

```
    put32(AUX_MU_CNTL_REG,0);               //Disable auto flow control and disable receiver and transmitter (for now)
```
这，在完成配置之前，我们先禁用接收器和发送器。我们也永久地禁用自动流控，因为它需要我们使用额外的GPIO引脚，而且 TTL-to-serial 转接线也不支持这样做。想要了解更多关于自动流控的内容，可以看[这篇文章](http://www.deater.net/weave/vmwprod/hardware/pi-rts/)。

```
    put32(AUX_MU_IER_REG,0);                //Disable receive and transmit interrupts
```
可以配置每当有新的数据时 Mini UART 就产生一个处理器中断。我们会在第三节课开始学习中断，所以我们暂时先禁用这个功能。

```
    put32(AUX_MU_LCR_REG,3);                //Enable 8 bit mode
```
Mini UART 可以支持7位或8位的操作。这是因为一个ASCII字符由7位编码表示，8位为了扩展。我们使用8位模式。

```
    put32(AUX_MU_MCR_REG,0);                //Set RTS line to be always high
```
RTS线路用于流量控制，我们不需要它，将它一直设置为高电平。
```
    put32(AUX_MU_BAUD_REG,270);             //Set baud rate to 115200
```
波特率是指信息在通道中传输的速率。“115200 baud” 表示串口每秒最高可传输115200位。树莓派 mini UART 设备的波特率应该和你的仿真终端的波特率一样。
Mini UART 根据下面公式计算波特率：
```
baudrate = system_clock_freq / (8 * ( baudrate_reg + 1 )) 
```
如果 `system_clock_freq` 是250 MHz，那么我们易得 `baudrate_reg` 为 270。

``` 
    put32(AUX_MU_CNTL_REG,3);               //Finally, enable transmitter and receiver
```
这行执行后，Mini UART 已经就绪！

### Sending data using the Mini UART（使用 Mini UART 发送数据）

当 Mini UART 就绪后，我们可以尝试用它发送和接收一些数据。我们可以使用下面两个函数来做：

```
void uart_send ( char c )
{
    while(1) {
        if(get32(AUX_MU_LSR_REG)&0x20) 
            break;
    }
    put32(AUX_MU_IO_REG,c);
}

char uart_recv ( void )
{
    while(1) {
        if(get32(AUX_MU_LSR_REG)&0x01) 
            break;
    }
    return(get32(AUX_MU_IO_REG)&0xFF);
}
```

两个函数都会启动一个无限循环，这样做的目的是要验证设备是否已经准备好发送或接收数据。我们用 `AUX_MU_LSR_REG` 寄存器来验证。第0位如果是1，表示数据已就绪；这意味着我们可以从UART读取数据。第5位如果是1，告诉我们发送器已空，意味着我们可以写数据到 UART。
下一步, 我们用 `AUX_MU_IO_REG` 寄存器存储已经发送的字符或者读取已经接收到的字符。

我们还有一个非常简单的函数，可以发送字符串而不是字符：

```
void uart_send_string(char* str)
{
    for (int i = 0; str[i] != '\0'; i ++) {
        uart_send((char)str[i]);
    }
}
```
这个函数只是迭代字符串中所有的字符，并逐个发送。

### Raspberry Pi config（树莓派的配置）

下面是树莓派的启动序列（简化版）：

1. 设备上电。
1. GPU 启动并从boot分区读取 `config.txt` 文件。这个文件含有GPU用来进一步调整启动序列的配置参数。
1. 将 `kernel8.img` 加载到内存并执行。

为了能执行我们的简单OS，`config.txt` 应该是下面这样：

```
kernel_old=1
disable_commandline_tags=1
```
* `kernel_old=1` 指定在0地址处加载内核映像。
* `disable_commandline_tags` 指示GPU不要向启动的映像传递任何命令行参数。


### Testing the kernel（测试内核）

现在我们已经阅读了所有的源码，是时候看它怎么运行了。你需要按照下面的步骤构建并测试内核：

1. 执行[src/lesson01](https://github.com/s-matyukevich/raspberry-pi-os/tree/master/src/lesson01)中的 `./build.sh` or `./build.bat` 构建内核
1. 将生成的 `kernel8.img` 文件复制到你的树莓派闪存卡的 `boot` 分区，然后删除你的SD卡中 `kernel7.img` 和其他任何类似 `kernel*.img` 的文件。确保你没有触及引导分区中的所有剩余文件。(查看 [43](https://github.com/s-matyukevich/raspberry-pi-os/issues/43) 和 [158](https://github.com/s-matyukevich/raspberry-pi-os/issues/158) 详细问题)
1. 按照前一节描述的内容修改 `config.txt` 文件。 
1. 按照[先决条件](../Prerequisites.md)的描述连接 USB-to-TTL 串口线。
1. 启动树莓派（上电）。
1. 打开你的仿真终端，你应该能在那看到 `Hello, world!` 消息。

注意：以上步骤是在假设你已经将Raspbian安装到你的SD上的前提下描述的。也可以用空的SD卡运行 PRi OS。

1. 准备SD卡:
    * 使用MBR分区表
    * 以FAT32模式格式化boot分区
    > 卡的格式应与安装Raspbian的要求完全相同。在[官方文档]中(https://www.raspberrypi.org/documentation/installation/noobs.md) 查看 `HOW TO FORMAT AN SD CARD AS FAT` 小节了解更多信息.
1. 复制下面的文件到卡中：
    * [bootcode.bin](https://github.com/raspberrypi/firmware/blob/master/boot/bootcode.bin) 这是GPU引导加载程序，它包含了GPU的启动和加载GPU固件的代码。
    * [start.elf](https://github.com/raspberrypi/firmware/blob/master/boot/start.elf) 这是GPU的固件。它读取 `config.txt` 并使GPU从 `kernel8.img` 中加载并运行 ARM指定的用户代码。
1. 复制 `kernel8.img` 和 `config.txt` 文件.
1. 连接USB-to-TTL串口线。
1. 启动树莓派（上电）。
1. 用你的仿真终端连接RPi OS。

不幸的是，所有的树莓派固件文件都是闭源的和不可读的。了解更多关于树莓派的启动序列，你可以查看一些非官方的文件，比如[这篇](https://raspberrypi.stackexchange.com/questions/10442/what-is-the-boot-sequence) 堆栈交换问题，或[这篇](https://github.com/DieterReuter/workshop-raspberrypi-64bit-os/blob/master/part1-bootloader.md) 引导加载程序。

##### Previous Page（上一页）

[Prerequisites](../../docs/Prerequisites.md)

##### Next Page（下一页）

1.2 [Kernel Initialization: Linux project structure](../../docs/lesson01/linux/project-structure.md)
