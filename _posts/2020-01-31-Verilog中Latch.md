---
layout:     post
title:      Verilog中Latch
subtitle:   
date:       2020-01-31
author:     yzmb2411
header-img: 
catalog: 	 true
tags:
    - FPGA
    - verilog 
---

今天在一个交流群中看到一个assign语句生成Latch的问题。下面是两个实例

示例1

```verilog
module test
(
	input	a,
	input	b,
	output	c
);

assign	c	=	(a & b) + (!b & c);
endmodule
```

elaborate结果

![image](C:/Users/pc/Documents/GitHub/yzmb2411.github.io/img/latch1.png)

Vivado工具elaborate之后，可以看出上述代码是一个纯组合逻辑，满足逻辑关系

c = (a & b) + (!b & c)  

synthesis结果

![image](C:/Users/pc/Documents/GitHub/yzmb2411.github.io/img/latch2.png)

Vivado工具综合之后，将设计映射到一个LUT3上。

示例2

```verilog
module test2
(
	input	a,
	input	b,
	output	c
);

	assign c = b?a:c ;
   
endmodule

```

elaborate结果

![image](C:/Users/pc/Documents/GitHub/yzmb2411.github.io/img/latch3.png)

Vivado工具elaborate之后，可以看出上述代码是一个Latch逻辑，满足逻辑关系

c =b?a:c ;

synthesis结果

![image](C:/Users/pc/Documents/GitHub/yzmb2411.github.io/img/latch4.png)

造成上述两个相同逻辑，不同综合结果差异的根本原因是：过程赋值和连续赋值的差异。

过程赋值在Verilog中主要用来赋值给reg变量，用来生成时序和组合逻辑。例如

always@(*)

连续赋值主要用来赋值给wire变量，用来生成组合逻辑。例如

assign

特别的是，Verilog中的

assign c =b?a:c ;

是一个特例。其赋值行为等价于

always@(b) begin

c = b?a:c ;

end

也就是说，其赋值行为其实是过程赋值

1、先计算出右边的值RHS

2、在将RHS赋值给左边的值LHS
 
而assign out = (a & b) + (!b &out)  

的赋值行为是连续赋值

计算出右边的值RHS，同时将RHS赋值给左边的值LHS。
 
最后得到的结论为
 
由过程赋值（always@和？=）建模的组合逻辑是否会生成锁存器，其根本原因是该组合逻辑存在保持功能！










