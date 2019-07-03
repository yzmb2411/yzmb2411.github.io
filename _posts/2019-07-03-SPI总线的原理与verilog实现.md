---
layout:     post
title:      SPI总线的原理与verilog实现
subtitle:   
date:       2019-07-03
author:     yzmb2411
header-img: 
catalog: 	 true
tags:
    - FPGA
    - 无线通信 
---

## 原理介绍

SPI（Serial Peripheral Interface，串行外围设备接口），是Motorola公司提出的一种同步串行接口技术，是一种高速、全双工、同步通信总线，在芯片中只占用四根管脚用来控制及数据传输，广泛用于EEPROM、Flash、RTC（实时时钟）、ADC（数模转换器）、DSP（数字信号处理器）以及数字信号解码器上。SPI通信的速度很容易达到好几兆bps，所以可以用SPI总线传输一些未压缩的音频以及压缩的视频。

下图是只有2个chip利用SPI总线进行通信的结构图

![image](https://wx1.sinaimg.cn/mw1024/ab20a024ly1g4mjuglz1zj20dm05at93.jpg)

可知SPI总线传输只需要4根线就能完成，这四根线的作用分别如下：

SCK(Serial Clock)：SCK是串行时钟线，作用是Master向Slave传输时钟信号，控制数据交换的时机和速率；

MOSI(Master Out Slave in)：在SPI Master上也被称为Tx-channel，作用是SPI主机给SPI从机发送数据；

CS/SS(Chip Select/Slave Select)：作用是SPI Master选择与哪一个SPI Slave通信，低电平表示从机被选中(低电平有效)；

MISO(Master In Slave Out)：在SPI Master上也被称为Rx-channel，作用是SPI主机接收SPI从机传输过来的数据；

SPI总线主要有以下几个特点：

1、采用主从模式（Master-Slave）的控制方式，支持单Master多Slave。SPI规定了两个SPI设备之间通信必须由主设备Master来控制从设备Slave。也就是说，如果FPGA是主机的情况下，不管是FPGA给芯片发送数据还是从芯片中接收数据，写Verilog逻辑的时候片选信号CS与串行时钟信号SCK必须由FPGA来产生。同时一个Master可以设置多个片选(Chip Select)来控制多个Slave。SPI协议还规定Slave设备的clock由Master通过SCK管脚提供给Slave，Slave本身不能产生或控制clock，没有clock则Slave不能正常工作。单Master多Slave的典型结构如下图所示

![image](https://wx3.sinaimg.cn/mw1024/ab20a024ly1g4mjugp0w2j20i20a977e.jpg)

2、SPI总线在传输数据的同时也传输了时钟信号，所以SPI协议是一种同步（Synchronous）传输协议。Master会根据将要交换的数据产生相应的时钟脉冲，组成时钟信号，时钟信号通过时钟极性(CPOL)和时钟相位(CPHA)控制两个SPI设备何时交换数据以及何时对接收数据进行采样，保证数据在两个设备之间是同步传输的。

3、SPI总线协议是一种全双工的串行通信协议，数据传输时高位在前，低位在后。SPI协议规定一个SPI设备不能在数据通信过程中仅仅充当一个发送者（Transmitter）或者接受者（Receiver）。在片选信号CS为0的情况下，每个clock周期内，SPI设备都会发送并接收1 bit数据，相当于有1 bit数据被交换了。数据传输高位在前，低位在后（MSB first）。SPI主从结构内部数据传输示意图如下图所示 

![image](https://wx1.sinaimg.cn/mw1024/ab20a024ly1g4mjugo8gej20na08n0uk.jpg)

SPI总线传输的模式：

SPI总线传输一共有4中模式，这4种模式分别由时钟极性(CPOL，Clock Polarity)和时钟相位(CPHA，Clock Phase)来定义，其中CPOL参数规定了SCK时钟信号空闲状态的电平，CPHA规定了数据是在SCK时钟的上升沿被采样还是下降沿被采样。这四种模式的时序图如下图所示：

![image](https://wx1.sinaimg.cn/mw1024/ab20a024ly1g4mjugv6sfj20nv0ba46h.jpg)

模式0：CPOL= 0，CPHA=0。SCK串行时钟线空闲是为低电平，数据在SCK时钟的上升沿被采样，数据在SCK时钟的下降沿切换

模式1：CPOL= 0，CPHA=1。SCK串行时钟线空闲是为低电平，数据在SCK时钟的下降沿被采样，数据在SCK时钟的上升沿切换

模式2：CPOL= 1，CPHA=0。SCK串行时钟线空闲是为高电平，数据在SCK时钟的下降沿被采样，数据在SCK时钟的上升沿切换

模式3：CPOL= 1，CPHA=1。SCK串行时钟线空闲是为高电平，数据在SCK时钟的上升沿被采样，数据在SCK时钟的下降沿切换

其中比较常用的模式是模式0和模式3。为了更清晰的描述SPI总线的时序，下面展现了模式0下的SPI时序图

![image](https://wx1.sinaimg.cn/mw1024/ab20a024ly1g4mjugofk0j20jt0beq5p.jpg)

上图清晰的表明在模式0下，在空闲状态下，SCK串行时钟线为低电平，当SS被主机拉低以后，数据传输开始，数据线MOSI和MISO的数据切换(Toggling)发生在时钟的下降沿(上图的黑色虚线)，而数据线MOSI和MISO的数据的采样(Sampling)发生在数据的正中间(上图中的灰色实线)。下图清晰的描述了其他三种模式数据线MOSI和MISO的数据切换(Toggling)位置和数据采样位置的关系图

![image](https://wx4.sinaimg.cn/mw1024/ab20a024ly1g4mjugoldhj20kf0atacs.jpg)

下面以模式0为例用Verilog编写SPI通信的代码。

## 目标任务

1、编写SPI通信的Verilog代码并利用ModelSim进行时序仿真

2、阅读Qual SPI的芯片手册，理解操作时序，并利用任务1编写的代码与Qual SPI进行SPI通信，读出Qual SPI Flash的Manufacturer/Device  ID

3、用SPI总线把存放在ROM里面的数据发出去，这在实际项目中用来配置SPI外设芯片很有用

## 设计思路与Verilog代码编写

SPI模块的接口定义与整体设计

Verilog编写的SPI模块除了进行SPI通信的四根线以外还要包括一些时钟、复位、使能、并行的输入输出以及完成标志位。其框图如下所示

![image](https://wx2.sinaimg.cn/mw1024/ab20a024ly1g4mjugmx2gj20h10fot9q.jpg)

其中：

　　I_clk是系统时钟；

　　I_rst_n是系统复位；

　　I_tx_en是主机给从机发送数据的使能信号，当I_tx_en为1时主机才能给从机发送数据；

　　I_rx _en是主机从从机接收数据的使能信号，当I_rx_en为1时主机才能从从机接收数据；

　　I_data_in是主机要发送的并行数据；

　　O_data_out是把从机接收回来的串行数据并行化以后的并行数据；

　　O_tx_done是主机给从机发送数据完成的标志位，发送完成后会产生一个高脉冲；

　　O_rx_done是主机从从机接收数据完成的标志位，接收完成后会产生一个高脉冲；

　　I_spi_miso、O_spi_cs、O_spi_sck和O_spi_mosi是标准SPI总线协议规定的四根线；

　　要想实现上文模式0的时序，最简单的办法还是设计一个状态机。为了方便说明，这里把模式0的时序再在下面贴一遍
