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
编译项目的时候, top 实体还是 `litecpu`, 不要改成 testbench 文件.

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

