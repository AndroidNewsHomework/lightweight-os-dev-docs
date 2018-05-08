此文档包含操作系统大作业中的备忘事项.

------------------------------------------------------------------------------

# 板子型号问题

## 概述
向勇给我们的板子自带的那张纸是错的, 然后好像 stepfpga 官网上的 doc 也有错.
我们板子上的 fpga 不是 10m02scm153i7g 而应该是 10m08scm153i7g

(其实余下不用看了) 后来我发现这就是他们的 docs / 纸片写错了.

    STEP-MAX10是小脚丫平台基于Altera公司芯片开发的FPGA开发板。核心FPGA芯片采用了Altera公司MAX10系列下的10M08SCM153/10M02SCM153。

注意选器件不要选错就好.

## 复现问题
使用最简单的 LED 闪烁的 helloworld, 程序如下
```
library IEEE;
use IEEE.STD_LOGIC_1164.ALL;
use IEEE.NUMERIC_STD.ALL;


entity Helloworld is
    port (
        clk: in std_logic;
        disp: out std_logic
     );
end Helloworld;


architecture behave of Helloworld is
    signal t_count: unsigned(21 downto 0);
    signal l_disp: std_logic;
    constant KT_COUNT_ZERO: unsigned(21 downto 0) := (others=> '0');
begin

    disp <= l_disp;

    process (clk) begin
        if (rising_edge(clk)) then
            t_count <= t_count + 1;
            if (t_count = KT_COUNT_ZERO) then
                l_disp <= not l_disp;
            end if;
        end if;
    end process;

end behave;
```

按照 stepfpga 给出的 max10 系列文档 [链接](http://www.stepfpga.com/doc/step-max10).
主要按照[软件文档](http://www.stepfpga.com/doc/_media/step-max10_v2%E8%BD%AF%E4%BB%B6%E6%89%8B%E5%86%8Cv1.0.pdf)建立项目, 使用上面的简单程序.
综合成功后写入 FPGA 时出现问题, 好象是 JTAG ID 出错的信息,
什么 expected jtag id 031810DD, found jtag id 031820DD.

## 如果复现失败
如果写入 fpga 时出现了其他错误, 尤其是什么 errorcode 89, 那么

* 检查 usb blaster 驱动. Linux 系统下执行 `lsusb` 看是否有 Altera 的设备.

* 检查当前用户是否有权限访问 usb 接口
Linux 系统下需设置 /etc/udev/rules.d/, [参考链接](https://ask.fedoraproject.org/en/question/82315/altera-quartus-usb-blaster-jtagd-udev-usb-permissions/).

## 解决问题

进入 quartus 的 bin 文件夹, 其中有一个文件 jtagconfig, 执行后得到类似如下输出
```
hob:bin/ $ ./jtagconfig
1) USB-Blaster [2-1]
  031820DD   10M08SA(.|ES)/10M08SC
```

没有发现 10m02 的设备, 在结合 stepfpga 的软件文档, 发现应该是第 11 页中 "6. 器件选择" 中出错.

之后将设备类型改为 10m08scm153i7g 重新编译 (Processing \> Start Compilation),
写入 fpga, 发现错误解决, 同时 helloworld 工作正常.

认为错误解决.

------------------------------------------------------------------------------

# RAM 的使用

## RAM 初始化
我们的板子不支持烧写 RAM 的初始化 (就是不能给一个 mif 文件作为 RAM 的初始值),
[参考](https://www.alteraforum.com/forum/showthread.php?t=56869).

------------------------------------------------------------------------------

# 片内Flash 的使用
尽管经过一系列的努力还是没有调通flash，不过把已经做过的事和问题先列在这里。
更新：现在尝试着跑一下官方的flash使用代码，还是跑不通，问题记录在这一个的部分的最后 *另一个尝试* 了。

## 添加Flash的QSys system
在quartus右边，IP catalog里选择install IP -> Library -> On Chip Memory -> Altera On-Chip Flash，双击打开QSys编辑页面，设置路径及文件名，按照需要配置一下时钟周期，突发模式等。

如果需要初始化Flash，勾选Flash Initialization中的Initialize flash content，然后根据需要选择从哪里读入flash的初始化内容。

选择右下角Generate HDL...，保存成VHDL或者Verilog文件。

## 把sof文件转换为pof文件
在File -> Convert Programming Files里可以进行。下面参考文献里的第一个链接说的很详细。

这里我没懂的是这个地方也有个初始化UFM的配置，不知道和之前那个Initialize flash content有什么区别

## 编写代码
没啥说的，例化flash

## 现在的情况
目前看来，flash似乎完全不能正常使用，至少不管是我手动写入数据还是初始化，都没有办法正常的读出来，读出来的似乎总是0。而且在最后的测试中我发现readValid似乎总是在高电平上，即使我手动的把dataread置为'0'。

参考文献的第二个链接是这个片内flash的手册，我按照他的例子给出了几个电平输入，全部gg。

而且有一些地方的设置，在device的configuration里，有些设置我们没有，很奇怪，不知道会不会和板子上片内flash的型号有关系。

## 可能的问题
* 我测试flash的代码写错了
* 对照教程配置flash的时候出错了
* 这个板子的片内flash比较特殊，教程无效
* 板子坏了（可能性极低）

## 另一个尝试
向老师给我们提供了官方的[flash演示项目](http://stepfpga.com/doc/fpga%E9%A1%B9%E7%9B%AE_%E4%BD%BF%E7%94%A8arduino%E7%8E%A9%E8%BD%ACfpga)，以下简称*官方项目*。

这个项目的flash是嵌入了51软核([PulseRain FP51-1T MCU](https://www.pulserain.com/fp51))的，依据我现在对软核的理解，应该是一个提供fpga与外设通讯平台的东西。按照图示来看fpga使用片内flash本来应该就不需要经过软核，官方也回复说要给我们单独flash的代码了。

在这个项目里on-chip flash部分的代码似乎是用system verilog书写的，这个代码我不太熟悉，还没有仔细看。

我在跑官方项目的过程中所做的尝试与出现的问题列举如下：
* 准备工作已经完成，下载了项目代码，quartus本来就有，版本比他高一点，也安装了Arduino IDE。这个Arduino是一个用高级语言（我看到的是C，据说有java的部分）完成硬件功能的开发平台，由Arduino和Arduino IDE分别提供硬件和软件支持，可以用高级语言指示开发板要做什么样的事情。
* 硬件逻辑那一块，quartus的项目是综合好的，可以直接烧入板子里，问题是现在也只能用它综合好的东西，因为似乎我没有办法再次编译综合（提示缺少库文件，我的确找不到他include的两个库文件，估计是没有公开的部分？但是我之前在git更新他submodule的时候也出现了问题，不知道有没有关系）
* Arduino的环境按照他说的已经配好了，很顺利，但是没有办法编译，编译时会报错。

**更新：我在写这个memo的时候本来打算去把报错信息复制一份过来的，然后不小心把文件删了，然后重新复制了一份项目代码，重新命了个名，结果又编译过了。请等待进一步的研究**

## 参考文献：
[片内flash参考手册](https://www.altera.com/documentation/vgo1395753117436.html)

[quartus片内flash教程](http://jingyan.eeboard.com/article/74039)


------------------------------------------------------------------------------

# Testbench 的使用
commit 3aadb63 给出了一个简单的包含 ram 的 testbench 例子.

## 基本使用
首先建立 testbench 文件并加入项目,
```vhdl
library IEEE;
use IEEE.STD_LOGIC_1164.ALL;
use IEEE.NUMERIC_STD.ALL;


entity litecpu_tb is
end litecpu_tb;


architecture behave of litecpu_tb is
  signal CLK: std_logic := '0';
  signal disp: std_logic := '0';
begin
  CLK <= not CLK after 10ns;

  utop:
  entity work.litecpu
  port map (
    clk=> CLK,
    disp=> disp
  );

end behave;
```
注意要求 vhdl 项目代码规范使用 tab 缩进而非空格

之后在 quartus 中,
Assignments
> Settings
> EDA tool settings : Simulation
> NativeLink settings
> compile test bench
中新建一个 testbench, 其中包括我们的 testbench 文件.

之后编译项目 (compile `Ctrl+L` 或者 start analysis and synthesis `Ctrl+K`.

之后
Tools
> Run Simulation tools
> RTL Simulation (行为仿真) 会打开 modelsim.

## modelsim 的使用
一般 RTL Simulation 打开的 modelsim 之后, 我们还需要调整信号等等.

下图是 modelsim 的界面

![Modelsim interface](https://github.com/AndroidNewsHomework/lightweight-os-dev-docs/blob/master/modelsim.png)

其中
* A 按钮是重新开始模拟, 时间回到 0 ps. 一般修改要观察的信号表之后需要点这个.
* B 框中填写你要运行的时间
* C 按钮使得时间前进, 前进长度为 B 按钮中时间长度

### modelsim 相关问题

一般我们使用的模拟器是 altera-modelsim, 注意是 starter version 才是免费的.
如果出现了 license 错误, 检查是否你的模拟器路径是正确的.

1. Assignments > Settings > EDA tool settings : Simulation > Tool name 需要是 modelsim-altera

2. Tools > Options > EDA Tools Options 中检查 modelsim-altera 的路径, 应当形如 `/home/hob/intelFPGA_lite/17.1/modelsim_ase/linuxaloem/`

如果出现了 unknown expanded name 之类的问题, 这是因为 modelsim 只负责仿真, 你所有的其他实体都需要已经被编译.
这种时候可以尝试修改 top 实体, 如改成 `CPU_CORE`.


------------------------------------------------------------------------------

# Cpu0 项目的结果重现

只叙述在 Linux (Ubuntu 16.04LTS) 环境下.

## Cpu0 项目概述, 以及前期准备
[Cpu0 项目](http://jonathan2251.github.io/lbd/) 是一个台湾人针对他的一个教学指令集 Cpu0
用 llvm 写了后端, 在此基础上写了教程.

前期准备主要有两件事情

1. 获取 Cpu0 项目代码: [下载链接](http://jonathan2251.github.io/lbd/lbdex.tar.gz)

2. 获取 llvm 3.9.0 代码 (版本很重要): [下载链接](http://releases.llvm.org/3.9.0/llvm-3.9.0.src.tar.xz)

另外, 本项目 (recc) 需要使用 clang 和 clang++. 可以通过
```
$ sudo apt-get install clang
```
安装.

## 复现 Cpu0 项目

1. 在 llvm 中注册 Cpu0 后端
```
$ cp LBDEX_PATH/src/modify/src/* LLVM_PATH -r
```

2. 复制 Cpu0 后端代码
```
$ cp LBDEX_PATH/Cpu0 LLVM_PATH/lib/Target/Cpu0 -r
```

3. 修改 `LLVM_PATH/CMakeLists.txt`, 删去不需要的目标, 只保留 Mips 和 Cpu0
```
set(LLVM_ALL_TARGETS
  Mips
  Cpu0
  )
```

4. 开始构建
```
$ cd LLVM_PATH
$ mkdir build
$ cd build
$ cmake .. -G "Unix Makefiles" -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_C_COMPILER=clang -DCMAKE_BUILD_TYPE=Debug
```
若成功, 最后几行输出应当是
```
-- Targeting Mips
-- Targeting Cpu0
-- Configuring done
-- Generating done
-- Build files have been written to: LLVM_PATH/build
```
之后执行
```
$ make -j4
```
即可, 其中 `-j4` 是 4线程并行, 对于更好的机器可以提高并行度.

  在最后链接生成目标可执行文件时, 可能出现机器内存不够的情况, 造成卡死.
  这时可以使用 `Ctrl-C` 手动终止 `make`, 之后单线程重新启动.
  我个人的方法是, 感觉电脑变卡之后检查负载 (使用`htop`), 如果内存负载过高就重新单线程启动`make`.

5. 验证结果

```
$ cd LLVM_PATH
$ cd build/bin
$ ./llc --version
```
输出中 `Registered Targets:` 中应当有 Mips 和 Cpu0.

之后新建 `foobar.c`, 内容为
```C
int main()
{
  return 0;
}
```

执行
```
$ clang -target mips-unknown-linux-gnu -c foobar.c -emit-llvm
$ clang -target mips-unknown-linux-gnu -S foobar.c -emit-llvm
```
分别生成 `foobar.bc` 和 `foobar.ll`, 是 LLVM 的中间表示.
可以加上优化选项 `-O1` 等优化代码大小和效率.

之后
```
$ ./llc -march=cpu0 -relocation-model=pic -filetype=asm foobar.bc -o -
```

可以看到输出了 Cpu0 的汇编代码, 以下是一个实例
```
        .text
        .section .mdebug.abiO32
        .previous
        .file   "foobar.bc"
        .globl  main
        .p2align        2
        .type   main,@function
        .ent    main                    # @main
main:
        .frame  $fp,8,$lr
        .mask   0x00001000,-4
        .set    noreorder
        .set    nomacro
# BB#0:
        addiu   $sp, $sp, -8
        st      $fp, 4($sp)             # 4-byte Folded Spill
        move     $fp, $sp
        addiu   $2, $zero, 0
        st      $2, 0($fp)
        move     $sp, $fp
        ld      $fp, 4($sp)             # 4-byte Folded Reload
        addiu   $sp, $sp, 8
        ret     $lr
        nop
        .set    macro
        .set    reorder
        .end    main
$func_end0:
        .size   main, ($func_end0)-main


        .ident  "clang version 3.8.0-2ubuntu4 (tags/RELEASE_380/final)"
        .section        ".note.GNU-stack","",@progbits
```

以上步骤均正确完成, 则 Cpu0 项目复现成功.

## 加速构建过程
主要可以通过使用更好的编译器和链接器来就加速构建 (`g++` 速度不够).
编译器使用 `clang` 和 `clang++` 的方法如前所描述.
链接时间事实上也占了不少, 因此下述链接器使用 llvm 项目的 `lld` 的方法.

通过 cmake 选项设置链接器十分麻烦, 因此我们直接修改 `ld`.

执行如下命令
```
$ sudo apt-get install lld-5.0  # 5.0 也可以是其他方法
$ file /usr/bin/ld        # 备份一下信息
$ sudo rm /usr/bin/ld
$ sudo ln -s /usr/bin/ld.lld-5.0 /usr/bin/ld
```

之后直接 `make` 即可.

事实上我尝试了三种链接器, 效果分别如下 (构建 `libSTO.so` 为例)

* 使用 `/usr/bin/ld.lld-5.0`:
  `make LTO  9.61s user 2.62s system 18% cpu 1:07.66 total`
* 使用 `/usr/bin/ld.gold`:
  `make LTO  15.16s user 3.43s system 28% cpu 1:04.10 total`
* 使用 `/usr/bin/ld.bfd` (gnu ld 默认):
  `make LTO  29.89s user 6.27s system 28% cpu 2:05.77 total`

所以还是 `lld` 最快, 而且它也比默认的 `ld.bfd` 省内存.

------------------------------------------------------------------------------

# llvm 工具链的准备

Cpu0 项目复现中, 我们只是完成了一个编译器. 现在准备链接器, 所以需要准备整个 llvm 项目.

## 版本要求
因为我们编译器基于 Cpu0, 因此要求的版本是 llvm-3.9.0.
链接器也是基于 [Cpu0 exlbt](http://jonathan2251.github.io/lbt/) 的,
注意 Cpu0 文档中写错了, 文档中声称使用 lld-3.5.0, 实际应当使用 lld-3.9.0.

clang 我们直接使用系统包管理器给我们的 clang 即可 (`apt-get install clang`).

其余 `cmake`, `make` 等软件直接使用系统自带的版本即可 (ubuntu 16.04LTS 自带即可).

## 配置
1. 下载 lld-3.9.0, 放入 `LLVM_PATH/tools/lld` 中. 可以通过 `svn` 来简单的完成
```
LLVM_PATH/tools/ $ svn co http://llvm.org/svn/llvm-project/lld/tags/RELEASE_390/final/ lld  
```

2. 下载 `exlbt`, 解压后复制其 `lld/` 到 llvm 的 `tools/lld` 中
```
LLVM_PATH/ $ cp EXLBT_PATH/lld tools/lld -r
```

## 编译
同上, 在 `build/` 中使用 `cmake` 和 `make` 构建. 以下关闭了一些模块, 加速构建过程.

```
build/ rm * -rf		# just clear cmake caches
build/ $ cmake -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_C_COMPILER=clang -DCMAKE_BUILD_TYPE=Debug -DLLVM_INCLUDE_TESTS=OFF -DLLVM_INCLUDE_EXAMPLES=OFF -G "Unix Makefiles" ..
build/ $ make -j2 # change parallel factor according to your machine
```

编译成功后会在 `build/bin` 中出现 `lld` 文件. 在 Linux 下我们使用 `ld.lld`.

## 测试
```
build/bin/ $ mkdir src && cd src
```

在 `build/bin/src/` 中新建 `main.c` 和 `foobar.c`, 内容如下

```
// main.c
extern unsigned a;

int main()
{
	a = 5;
	return 0;
}


// foobar.c
unsigned int a = 6;
```

之后
```
src/ $ clang -target mips-unknown-linux-gnu -O0 -emit-llvm -c main.c # generates main.bc
src/ $ clang -target mips-unknown-linux-gnu -O0 -emit-llvm -c foobar.c # generates foobar.bc
src/ $ ../llc -march=cpu0 -relocation-model=static -filetype=obj main.bc -o main.o
src/ $ ../llc -march=cpu0 -relocation-model=static -filetype=asm main.bc -o main.s
src/ $ objdump -s main.o > main.rel.od.s
src/ $ ../llc -march=cpu0 -relocation-model=static -filetype=obj foobar.bc -o foobar.o
src/ $ ../llc -march=cpu0 -relocation-model=static -filetype=asm foobar.bc -o foobar.s
src/ $ objdump -s foobar.o > foobar.rel.od.s
src/ $ ../ld.lld *.o -e main 
src/ $ objdump -s a.out > a.out.exe.od.s
```

生成文件中, 我们关注 
1. `main.s`, `foobar.s`: 生成的汇编代码
2. `main.rel.od.s`, `foobar.rel.od.s`: 生成的可重定向文件中的内容
3. `a.out.exe.od.s`: 生成的可执行文件中的内容

观察 `*.rel.od.s`, 发现
* `*.rel.od.s` 中指令和代码的地址都是从 `0000` 开始
* `main.rel.od.s` 中载入变量 `a` 地址的两条指令中 
(`lui` 和 `ori`, 后续版本可能是 `lui` 和 `addiu`)
`a` 的地址都是 `00000000`. 在可重定向目标文件里, 这表示 `a` 的地址还没确定

观察 `a.out.exe.s`, 发现
* 指令从 `0x20000` 开始, 数据从 `0x30000` 开始. 这是默认值, 可以通过链接脚本修改.
* 载入变量 `a` 地址的两条指令中, `a` 的地址确定到了 `0x00030000`.
