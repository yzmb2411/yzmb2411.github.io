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











