# 使用爱克斯板与CODESYS实现软PLC配置并与外界程序通信。

## Use the AIxboard with CODESYS to implement soft PLC configuration and communicate with external programs.

## 1 序言
### 1.1  爱克斯板介绍
AIxBoard™爱克斯板开发者套件是一款功能强大的小型计算机，专为支持入门级边缘人工智能应用程序和设备而设计。无论是在人工智能学习、开发还是实训等应用场景下，它都能完美胜任。
该开发板是类树莓派的x86主机，可支持Linux Ubuntu及 完整版Windows操作系统。板载一颗英特尔4核处理器，最高运行频率可达2.9 GHz，且内置核显（iGPU），板载 64GB eMMC存储及LPDDR4x 2933MHz（4GB/6GB/8GB），内置蓝牙和Wi-Fi模组，支持USB 3.0、HDMI视频输出、3.5mm音频接口，1000Mbps以太网口。完全可把它作为一台mini小电脑来看待，且其可集成一块Arduino Leonardo单片机，可外拓各种传感器模块。
此外, 其接口与Jetson Nano载板兼容，GPIO与树莓派兼容，能够最大限度地复用树莓派、Jetson Nano等生态资源，无论是摄像头物体识别，3D打印，还是CNC实时插补控制都能稳定运行。可作为边缘计算引擎用于人工智能产品验证、开发；也可以作为域控核心用于机器人产品开发。
然而，虽然爱克斯板具有如上的诸多优点。但由于其运行的Windows或者Linux系统都是非实时性的操作系统，难以用于对实时性要求较高的工业环境中。而实时性的PLC环境通常较为封闭，难以使用python，Openvino等外界程序。

### 1.2  CODESYS介绍
CODESYS是一款工业自动化领域的一款开发编程系统(CODESYS是Code System的简写)，应用领域涉及工厂自动化、汽车自动化、嵌入式自动化、过程自动化和楼宇自动化等等。CODESYS软件可以分为两个部分，一部分是运行在各类硬件中的RTE（Runtime Environment），另一部分是运行在PC机上的IDE。因此CODESYS的用户既包括生产PLC、运动控制器的硬件厂商，也包括最终使用PLC、运动控制器的用户。
目前全球有近400家的控制系统生产制造商是CODESYS的用户：如ABB、施耐德电气SchneiderElectric、伊顿电气EATON、博世力士乐Rexroth、倍福BECKHOFF、科控KEBA、日立HITACHI、三菱自动化MITSUBISHI、欧姆龙OMRON、研华科技、凌华科技ADLINK、新汉电脑、和利时集团、SUPCON 中控集团、步科自动化KINCO、深圳雷赛、汇川技术、深圳合信、深圳英威腾、华中数控、固高科技等等。
简单来说，CODESYS可以说是PLC界的安卓，许多PLC厂商都以CODESYS作为其PLC的内核。
此外，CODESYS可以将任何一款arm架构或者x86架构的处理器变为实时的PLC系统。CODESYS结合AIxBoard，我们能够得到一个可以用于工业控制检测领域的一款功能强大的人工智能小型计算机。

## 2 前期准备

CODESYS软件分三层架构，可用下图来表示：

图1  CODESYS软件架构示意图
其中开发层(IDE)可使用CODESYS Development System(具有完善的在线编程和离线编程功能)、编译器及其配件组件、可视化界面编程组件等对CODESYS程序进行开发与部署。本文使用的版本为CODESYS V3.5 SP17，下载与安装教程可见：CODESYS 3.5.17.0 软件安装_codesys安装教程_小 Co的博客-CSDN博客。

### 2.1  开发层主机前期准备

在安装完CODESYS后，还需要根据需求下载安装部分CODESYS软件包，由于本文需要在运行有Ubuntu的AIxBoard上部署CODESYS Runtime，并通过共享内存实现与外界程序通信，故需安装的软件包有以下几种：

1．CODESYS Control for Linux SL

2．CODESYS Edge Gateway for Linux

3．Shared Memory Communication

完成安装后，可在包管理器中查看到这三个软件包。

图2 在CODESYS中安装软件包

安装完成三个软件包后，重启CODESYS，随后能够在工具中最下面一行找到Update Linux，点击后会打开一个能够与安装了Linux系统的AIxBoard进行通信部署的界面。

图3 安装软件包完成后的效果

### 2.2  设备硬件层前期准备

为了提高AIxBoard的适用性，本文将使用Ubuntu系统作为AIxBoard的操作系统，系统版本为Ubuntu 20.04LTS，这里使用的是Canonical为Intel优化的版本。下载与安装教程如下：系统安装 - AIxBoard开发指南 (xzsteam.com)。
除此之外，安装完成系统后，还需安装python以进行共享内存通信，本文使用的python版本为3.8.10。
为验证CODESYS能够与外界程序通信，同时也安装了Epics。Epics全称为Experimental Physics and Industrial Control System即“实验物理及工业控制系统”，是上世纪90年代初由美国洛斯阿拉莫斯国家实验室（LANL）和阿贡国家实验室（ANL）等联合开发的大型控制软件系统。安装完成Epics后，需使其在后台运行，后续将通过CODESYS与其进行通信。

图4 在AIxBoard中预先安装好Ubuntu系统与Epics

## 3 工程建立

### 3.1  新建标准工程

在CODESYS中，选择文件-新建工程，命名工程为AIxBoard，选择新建标准工程。

图5 新建标准工程
在弹出的标准工程对话框中，选择设备为CODESYS Control for Linux SL，选择结构化文本(ST)作为编程语言。

图6 新建标准工程选项

### 3.2  加载所需函数库

将我们刚刚安装的软件包中的所需函数库加载到此工程中，需要添加的函数库有：

· SysShm,3.5.8.0 (System)

· SysTypes2 Interfaces,3.5.4.0 (System)

打开库管理器（Library Manager）,选择“添加库（Add Library）”,点“高级（Advanced...）”;



图7 在工程中加载刚刚安装好的函数库
在搜索框（String for a fulltext search...）中分别输入SysShm和SysTypes搜索添加SysShm,3.5.8.0 和SysTypes2 Interfaces,3.5.4.0 ,
选中搜索到的库，点“OK”确认添加，
       

图8  搜索并添加所需的两个函数库

### 3.3  建立设备通信
点击工具-Update Linux打开与Linux通信的界面，在左侧输入用户名和密码，搜索到AIxBoard的IP后，点击Install将CODESYS Runtime安装至AIxBoard中，安装文件可以在AIxBoard的/etc/中找到。

图9 与AIxBoard通信并将Runtime部署在AIxBoard上
经过图9的操作之后，AIxBoard便已经成为了一个能够运行CODESYS的实时性系统的PLC了。

图10 在AIxBoard上安装好的CODESYS Runtime程序文件
新建项目后，点击左下角设备进入设备树，双击Device后，点击扫描网络进行设备连接，选择AIxBoard为控制器的网络路径。

图11 进行设备扫描与连接
输入账号密码进行登录，如果是第一次登陆，还需要另外设置一次登录密码。

图12 在CODESYS Runtime上登录并自动下载代码
登陆完成后，将会自动下载程序代码至AIxBoard上，并且可以在device中看到设备信息。

图13 连接完成后的设备网络图

## 4 	代码编写

### 4.1  定义数据单元类型与全局变量
右击Application，选择添加DUT（Data Unit Type，数据单元类型），DUT为自定义的数据类型，本文中新建自定义的数据单元类型目的为通过不同类型的数据单元，将输出至外部程序的变量与从外部程序输入进来的变量分离开。
新建两个数据类型分别为：Str_ParaFromHMI与Str_ParaToHMI，目前结构体内部仅包含一个长整型格式的数据(LREAL)，可根据实际需求修改或添加。

	TYPE Str_ParaToHMI :
	STRUCT
	fOut: LREAL;
	END_STRUCT
	END_TYPE
	
	TYPE Str_ParaFromHMI :
	STRUCT
	fIn: LREAL;
	END_STRUCT
	END_TYPE

右击Application添加全局变量列表GVL(Global Var List)，并将刚刚新建的两种数据类型实例化，并添加至全局变量中。实例化的名称分别为GetPara与SetPara。其中GetPara用于从外部程序中获取数据进入CODESYS,SetPara用于将CODESYS中的数据输出至外部程序中。

	VAR_GLOBAL
		GetPara:Str_ParaFromHMI;
		SetPara:Str_ParaToHMI;
	END_VAR

### 4.2  编写共享内存POU

右击Application添加POU(Program organizational unit，程序组织单元)，命名为Sharedmemory。

图14 新增程序组织单元的相关配置

POU上方为局部变量声明区域，下方为结构化文本程序区域。
局部变量声明如下：

	PROGRAM SharedMemory
	VAR	
	bStart: BOOL:= FALSE;
	ReadHandle: RTS_IEC_HANDLE:= RTS_INVALID_HANDLE;
	WriteHandle: RTS_IEC_HANDLE:= RTS_INVALID_HANDLE;
	szNameRead: STRING:= 'CODESYS_MEMORY_READ';		//声明共享内存的读取内存名称
	szNameWrite: STRING:= 'CODESYS_MEMORY_WRITE';	//声明共享内存的写入内存名称
	ulPhysicalAddressRead: __UXINT:= 0;//读取数据的偏移地址，0为从头读取
	ulPhysicalAddressWrite: __UXINT:= 0;//写入数据的偏移地址，0为从头写入
	ulSizeRead: __UXINT:= 1024;//读取空间大小
	ulSizeWrite: __UXINT:= 1024;//写入空间大小
	ResultRead: ARRAY[0..2] OF RTS_IEC_RESULT;		//返回运行错误码,0中为运行错误码，1中为读取执行错误码，2中为写出执行错误码
	ResultWrite: ARRAY[0..2] OF RTS_IEC_RESULT;     //返回运行错误码,0中为运行错误码，1中为读取执行错误码，2中为写出执行错误码
	
	SMRead: __UXINT;
	SMWrite: __UXINT;
	ulOffsetRead: __UXINT:= 0;
	ulOffsetWrite: __UXINT:= 0;
	END_VAR

其中，高亮部分语句所指定的名称是之后需要与python中读取共享内存中数据一致的文件名称。可任意修改但是应与python中程序一致，共享内存的文件将会保存在/dev/shm/中。
下方ST程序部分编写代码如下：

```
//Init Memory
IF NOT bStart THEN
	ReadHandle:= SysSharedMemoryCreate(pszName:= szNameRead, ulPhysicalAddress:= ulPhysicalAddressRead, pulSize:= ADR(ulSizeRead), pResult:= ADR(ResultRead[0]));
	WriteHandle:= SysSharedMemoryCreate(pszName:= szNameWrite, ulPhysicalAddress:= ulPhysicalAddressWrite, pulSize:= ADR(ulSizeWrite), pResult:= ADR(ResultWrite[0]));
	IF RTS_INVALID_HANDLE <> ReadHandle AND RTS_INVALID_HANDLE <> WriteHandle THEN
		bStart:= TRUE;
	END_IF
END_IF

//读入数据
IF RTS_INVALID_HANDLE <> ReadHandle THEN
	SMRead:= SysSharedMemoryRead(
	hShm:= ReadHandle, 					//读取内存的设备句柄
	ulOffset:= ulOffsetRead,			//读取数据的偏移地址 
	pbyData:= ADR(GVL.GetPara), 		//指向读取数据的缓冲区
	ulSize:= SIZEOF(Str_ParaFromHMI), 	//读取数据的字节大小	
	pResult:= ADR(ResultRead[1]));		//返回执行的错误码
END_IF

//写出数据
IF RTS_INVALID_HANDLE <> WriteHandle THEN
	SMWrite:= SysSharedMemoryWrite(
	hShm:= WriteHandle, 				//写入内存的设备句柄
	ulOffset:= ulOffsetWrite, 			//写入数据的偏移地址
	pbyData:= ADR(GVL.SetPara), 		//指向写入数据的缓冲区
	ulSize:= SIZEOF(Str_ParaToHMI), 	//写入数据的字节大小
	pResult:= ADR(ResultWrite[2]));		//返回执行的错误码
END_IF
```

在Maintask中调用编辑好的POU，将此POU加入到执行程序中。

图15 在任务配置中调用编写好的程序
### 4.3  编写数据来源POU
在主程序PLC_RPG中添加正弦数据函数，不断向SetPara中发送正弦波数据。

图17 编写主程序相关函数，用于输入正弦波形

完成后，点击上方编译，编译通过后即可将程序登录下载至AIxBoard中。
在AIxBoard上，编写相关python程序接收来自CODESYS传递的信号并通过pyepics将其发送至Epics中，代码如下：

```python
import mmap
import struct
from epics import caput
import epics
import time
name="CODESYS_MEMORY_WRITE"
f= open('/dev/shm/'+name,"r")
while 1:
    f.flush()
    mm=mmap.mmap(f.fileno(),0,prot=mmap.PROT_READ)
    #print(mm.read(8))
    [number,]=struct.unpack('d',mm.read(8))
    print(number)
    #print(epics.ca.find_libca())
    caput('aiHost:xxxExample',number)
time.sleep(0.05)
```

## 5 	运行结果

以管理员身份运行python程序，可在AIxBoard上不断读取到CODESYS发送的数据。

图18 AIxBoard上最终运行结果，左侧为接收到的数据量
同时在CODESYS中可建立信号跟踪器，检测发送出的数据波形。

图19 信号跟踪器上显示的CODESYS中发出的数据波形
通过新建CS-Studio界面，可以从Epics中查看数据，验证CODESYS中发送出来的数据的正确性。

图20 在CS-Studio界面上监视到的Epics网络中PV量的变化波形
至此，我们已完成了将AIxBoard变为PLC并与外界程序通信的全部任务，顺利将AIxBoard从一台非实时性的开发板变成了一个能够用于工业控制领域的实时PLC控制器。能够与外界程序进行通信，使基于AIxBoard与CODESYS配置而成的软PLC相比传统的PLC而言，具有了更高的灵活性，通过搭配OpenVINO等人工智能模型，能够实现更加智能化的控制效果。
文中所涉及到的所有工程文件与代码均已开源于github，网址为：https://github.com/EHU0/Codesys_ShareMemory_On_AIxBoard.git
