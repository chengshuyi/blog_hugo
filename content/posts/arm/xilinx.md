---
title: "Xilinx"
date: 2020-04-29T17:34:47+08:00
description: ""
draft: true
tags: []
categories: []
---





1. 页面刷新多次（大概几十次）就打不开网页了；
2. 有时会出现unable to alloc pbuf in recv_handler错误问题；
3. 文件传输涉及到跨域的问题；





1. mem_malloc返回NULL指针，打开文件错误

2. 出现很多的open error问题



ERROR: unable to open file, Res = 9, name = /index.htm
ERROR: unable to open file, Res = 9, name = /404.html
ERROR: unable to open file, Res = 9, name = /404.htm
ERROR: unable to open file, Res = 9, name = /404.shtml
ERROR: unable to open file, Res = 9, name = /index.htm
ERROR: unable to open file, Res = 9, name = /404.html
ERROR: unable to open file, Res = 9, name = /404.htm
ERROR: unable to open file, Res = 9, name = /404.shtml
ERROR: unable to open file, Res = 9, name = /index.htm



### 2020年6月20日

1. oske新建一个10ms执行一次的任务，去检测状态；



axi的状态：

* IDLE：可以从axi发送数据或者接收数据
* BUSY ：不可以从axi发送数据或者接收数据

axi总线可以发送配置文件、测试文件、输入文件、清除文件、使能文件，但是每次只能完整的执行其中一个

配置文件的配置状态：

* IDLE：该文件还没有被配置；
* ING：正在配置，但是还没有完成；
* TIMEOUT：配置超时
* DONE：配置完成

备注：天杰的配置文件的结构体需要新增：配置文件指令个数、配置文件已发送指令个数、配置文件上个指令后等待返回的时间

输入文件的配置状态：

* IDEL：该文件还没有输入；
* ING：正在输入，但是还没有完成；
* TIMEOUT：输入超时；
* DONE：输入完成

测试文件可以看作是一个配置文件

### 2020年6月18日

问题：

1. 需要netmask gw mac_addr
2. master需要新增每一个板子的pcb
3. malloc slaves fail
4. f_mkfs(Path, FM_SFD, 0, work, sizeof work);
5. FF_MAX_SS
6. 测试400个是正常的，lwip大概在超时2小时候是75秒进行

### 2020年6月17日

1. 移植httpd client，目前仍存在一些问题，做了些测试：
    a. 采用127.0.0.1 环路测试，发现无法创建tcp连接；
    b. 采用192.168.1.10 进行自身ip测试，发现也无法创建tcp连接；
    c. pc端建立一个简单的http server，可以接收到请求数据，但是没有返回；
2. httpd client仍然存在一些问题：
    a. 无法通过post发送数据，移植的里面只有get方法；
    b. 建议采用tcp，这样的话比较方便，操作码也比较容易统一；

### 2020年6月11日

1. httpd post增加cgi功能；
2. httpd增加返回功能；

### 2020年5月14日

1. 解决文件传输问题，读/写入文件出现FR_INVALID_OBJECT问题，原因是FIL file全局变量没有初始化；
2. 文件名长度不能超过8个字符

### 2020年5月18日

解决input文件传输问题：

1. 先移植tcp传输Input file，发现仍然传输不了，怀疑是config配置文件有问题；
2. 测试是否配置文件发错时是否会影响input file的传输；（用戴书画原来的文件测试发现会）
3. 定位到config发送时出现的问题；


jtag不识别问题

### 2020年5月20日

1. 解决上下文切换后，程序跑飞的问题（重新移植osek）

### 2020年5月27日

![img](https://images2018.cnblogs.com/blog/867021/201803/867021-20180322001733298-201433635.jpg)

1. 解决多次Input出现卡死的问题（配置文件有问题）

### 2020年5月28日

1. 删除row文件，直接从input文件解析
2. 解决多次（几十次）input出现卡死的问题（重新生成输入文件）
3. 修改提取文件名函数，和师弟保持一致
4. 解决文件传输问题，由于匹配前缀，导致文件传输失败；
5. 解决文件名字问题，主要是memcpy后，没有在字符串末尾加'\0'

### 2020年6月2日

1. 修改整个工程的名字位darwin_os
2. 咨询戴书画相关硬件问题：
   * 可以修改神经元的时钟频率吗
     可以做，但是目前是写死的100m
   * 写入一次配置文件，会刷新整个芯片的存储空间吗
     不会，采用的是SRAM作为存储器
   * input文件转换过程
     需要知道配置文件的第一层映射了哪些神经元，50个文件每一个文件都包含了这张图片的所有像素信息。
   * 脉冲节拍是发送给整个芯片吗，脉冲的作用，通知芯片开始计算，tick拉低开始计算，中断会有结果返回。

### 2020年6月4日

1. 四个芯片作为一个子板，地址是扩充的，也就是地址编程单个芯片的四倍；
2. 子板之间的连接是通过添加一个虚拟的节点进行实现的；
3. 最后的配置文件时分开的；
4. 虚拟节点是48x16，一个节点有256个神经元；
5. 子板的规模是48x48
6. 单芯片的规模是24x24
7. 确保一个子板能够跑一层网络

首先，想到的两个文件就是 PL 部分需要的 bit 文件以及 PS 需要的 elf 文件。但是仅有这两个文件不够的。

我们还需要一段代码把 bit 文件以及 elf 文件安置好。这段代码就是大名鼎鼎的 FSBL.elf。

BOOT.bin = FSBL.elf+该工程.bit+该工程.elf

PS（processing system），就是与FPGA无关的ARM的SOC的部分；

PL（可编程逻辑），也就是FPGA部分；

**SD 固化**：将镜像文件拷贝到 SD 卡，设置拨码开关，使系统从 SD 模式启动。那么每次断电重启后，系统都会

从 SD 启动。

**QSPI FLASH** **固化**：设置拨码开关，将镜像文件烧写进 FLASH，使系统从 QSPI-FLASH 模式启动。那么每次

断电重启后，系统都会从 FLASH 启动。

**启动模式** 

**开关状态**

SD 启动/JTAG 调试模式 

开关 1-OFF，开关 2-OFF

QSPI FLASH 启动/JTAG 调试模式 

开关 1-ON ，开关 2-OFF

### BOOT.bin制作过程

step1：创建vivado工程，添加QSPI flash接口；添加SDIO接口；添加串口；

step2：导入SDK;

step3：创建应用工程；

step4：创建FSBL工程；

step5：右键应用工程，选择 Creat Boot Image

### SD-TF卡启动

将生成的 BOOT.bin（名字必须是这个，不然不识别） 文件，复制到 SD 卡，再将 SD 卡插到开发板，最后打开电源。则开机后系统从 SD 卡启动，程序掉电不消失。

### QSPI-FLASH 下载方法如下

Step1: 新建环境变量：计算机->属性->高级系统设置->高级->环境变量->新建系统变量

​	变量名：XIL_CSE_ZYNQ_UBOOT_QSPI_FREQ_HZ

​	变量值：10000000

Step2: 生成加载 QSPI FLASH 的 fsbl 文件：新建一个新的 FSBL 文件，命名为 zynq_fsbl。File->New->Application Project，输入 zynq_fsbl，点击 Next。选择Zynq FSBL，单击 Finish。

Step3: 打开 zynq_fsbl 的 main.c 文件，在此处增加`BootModeRegister = JTAG_MODE;`保存并编译

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200429174920.png)

Step4: 模式开关切换到 QSPI 启动模式（1-ON ,2-OFF），开发板通电。选择 Xilinx Tools > Program Flash 或单击

Program Flash Memory。

Step6:下载完成后断电，重新打开电源，就能看到从 QSPI FLASH 加载。





https://blog.csdn.net/maddisonn/article/details/88062796





工作记录：

1. debug应该使用右键项目debug as，不要使用快捷栏得按钮；
2. sd path的编号是逻辑编号，一般0是sd卡，1是emmc，但是我只是用了emmc，所以emmc的path变成了0；
3. f_mkfs是初始化整个emmc，重新写入fat表，所以调用一次后就可以注释掉了
4. 网络ping不通，修改为固定速率1000M
5. xilffs undefined reference to f_open  https://blog.csdn.net/weixin_44167319/article/details/104074045
