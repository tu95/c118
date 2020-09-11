---
title:   gsm嗅探(2)c118环境搭建

---

# 前提准备

OsmocomBB项目的所有文章都在这个地址 [https://osmocom.org/projects/baseband/wiki ](https://osmocom.org/projects/baseband/wiki)，这里详细记录了每一个细节，请自行研读，博主只写了环境搭建的部分。

MS：Mobile Station，移动终端；
IMSI：International Mobile Subscriber Identity，国际移动用户标识号，是TD系统分给用户的唯一标识号，它存储在SIM卡、HLR/VLR中，最多由15个数字组成；
MCC：Mobile Country Code，是移动用户的国家号，中国是460；
MNC：Mobile Network Code ，是移动用户的所属PLMN网号，中国移动为00、02，中国联通为01；
MSIN：Mobile Subscriber Identification Number，是移动用户标识；
NMSI：National Mobile Subscriber Identification，是在某一国家内MS唯一的识别码；
BTS：Base Transceiver Station，基站收发器；
BSC：Base Station Controller，基站控制器；
MSC：Mobile Switching Center，移动交换中心。移动网络完成呼叫连接、过区切换控制、无线信道管理等功能的设备，同时也是移动网与公用电话交换网(PSTN)、综合业务数字网(ISDN)等固定网的接口设备；
HLR：Home location register。保存用户的基本信息，如你的SIM的卡号、手机号码、签约信息等，和动态信息，如当前的位置、是否已经关机等；
VLR：Visiting location register，保存的是用户的动态信息和状态信息，以及从HLR下载的用户的签约信息；
CCCH：Common Control CHannel，公共控制信道。是一种“一点对多点”的双向控制信道，其用途是在呼叫接续阶段，传输链路连接所需要的控制信令与信息。

## 0x01.准备c118 

- MotorolaC123/C121/C118 (E88) — our primary target
- MotorolaC140/C139 (E86)
- MotorolaC155 (E99) — our secondary target
- MotorolaV171 (E68/E69)
- SonyEricssonJ100i
- Pirelli DP-L10
- Neo 1973 (GTA01)
- OpenMoko – Neo Freerunner (GTA02)
- SciphoneDreamG2 (MT6235 based)
  我们选择Moto C118，因为官方支持的最好、硬件成本低，￥35/台（手机+电池+充电器）
  ![p3](http://www.vuln.cn/wp-content/uploads/drops/20160106/201601061509262473932.jpg)

## 0x02. USB转串口模块

CH340,CP2102叽里呱啦的都是可以的，能正常识别就行。

## 0x03.系统要求

这是我的系统，64位的ubuntu16.0.4，内核版本4.15.0

```
Linux box 4.15.0-47-generic #50~16.04.1-Ubuntu SMP Fri Mar 15 16:06:21 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux

No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 16.04.6 LTS
Release:	16.04
Codename:	xenial
```

我的机器是skywangg@box，所以以下操作都是在/home/skywangg 目录下进行。

# 2.创建ARM编译环境（armtoolchain）

OsmocomBB项目是运行在一个移动终端也就是一个手机上面，生成了一个固件并刷入手机，用于控制手机的各种操作并提供一个接口给电脑接收电脑传过去的指令，那么就需要本地电脑有一个用于连接手机的通道，在这里叫做osmocon，制作手机固件需要使用ARM编译环境编译生成最终的.bin文件，所以首先就是搭建这个ARM编译环境。

开始搭建环境之前，我认为你已经准备好了Linux环境并更新到最新版本，创建一个文件夹用来存放ARM编译环境的所需文件，这里假设创建的文件夹为armtoolchain，进入文件夹后下载构建脚本并赋予其可执行权限

```
mkdir armtoolchain
cd armtoolchain
wget https://osmocom.org/attachments/download/2052/gnu-arm-build.3.sh
sudo chmod +x gnu-arm-build.3.sh
```

接着创建几个文件夹

```
mkdir build install src
```

这几个文件夹分别存放了对应的文件，切换到src文件夹下载必要的文件包

注意：这里的下载地址可能很慢或者无法下载，请自行准备好梯子

```
cd src/
wget https://ftp.gnu.org/gnu/gcc/gcc-4.8.2/gcc-4.8.2.tar.bz2
wget https://ftp.gnu.org/gnu/binutils/binutils-2.21.1a.tar.bz2
wget ftp://sources.redhat.com/pub/newlib/newlib-1.19.0.tar.gz
```

这个时候不要着急构建，先安装一些必须的依赖

```
sudo apt-get install build-essential libgmp3-dev libmpfr-dev libx11-6 libx11-dev texinfo flex bison libncurses5 libncurses5-dbg libncurses5-dev libncursesw5 libncursesw5-dbg libncursesw5-dev zlibc zlib1g-dev libmpfr4 libmpc-dev
```

然后就可以执行构建脚本了

```
cd ..
./gnu-arm-build.3.sh
```

按任意键开始执行构建，直到出现以下提示信息则构建成功

```
Build complete! Add /home/skywnagg/armtoolchain/install/bin to your PATH to make arm-none-eabi-gcc and friends
accessible directly.
```

根据提示信息把上面的路径添加到环境变量中，环境变量在/home/kbdancer/.bashrc，使用vim编辑然后将下面的这行添加到环境变量

```
export PATH=$PATH:home/skywangg/armtoolchain/install/bin
```

这里注意把路径改成你自己的，然后执行

```
source /home/kbdancer/.bashrc
```

让环境变量生效，接下来就可以编译osmocombb了。

# 3.编译osmocombb

开始之前需要安装一些必要的依赖包

```
sudo apt install libtool shtool automake autoconf git-core pkg-config make gcc
```

这些依赖包是编译必备的软件包，osmocomBB项目是基于libosmocore的，所以要先搞定 libosmocore，libosmocore的编译安装也有一些依赖需要安装

```
sudo apt-get install build-essential libtool libtalloc-dev shtool autoconf automake git-core pkg-config make gcc
```

接下来还必须安装一些库

```
sudo apt-get install libpcsclite-dev
```

下面就可以把代码同步下来了

```
git clone git://git.osmocom.org/libosmocore.git
```

同步代码之后就可以编译了

```
cd libosmocore/
autoreconf -i
./configure
make
sudo make install
sudo ldconfig -i
cd ..
```

编译安装完libosmocore就可以继续编译osmocombb了，首先同步最新代码（注意：请自备梯子）

```
git clone git://git.osmocom.org/osmocom-bb.git
cd osmocom-bb
git pull --rebase
```

代码拉取回来之后执行编译

```
cd src
make
```

遇到这些缺乏某些依赖的，根据报错，百度。

```
No package 'libpcsclite' found
No package 'gnutls' found
```

![](http://image.skywang.fun/picGO/20200215002723.png)

如果遇到红字这种(这张图片转载自安全客：https://www.anquanke.com/post/id/84627，当时我忘记截图了。)都是OsmocomBB交叉编译环境出现问题，因为网上很多文章说一半不说一半导致 没安装好标题二里面的环境包又或者你的环境包下载错了。这时候重新指向一下gcc编译环境目录。

```
export PATH=/home/skywangg/armtoolchain/install/bin:$PATH
实际路径根据你的机器来操作。
```

再有其他报错，就先复制报错第一天，复制粘贴搜索解决方法。

这篇文章有详细的改错方法:https://www.anquanke.com/post/id/84627

```
修改问题文件（如果你是gnu-arm-build.2.sh并且没有出现cell扫描不动的问题，请跳过这一步）

进入osmocom-bb找到这些文件并修改他们            
vi osmocom-bb/src/target/firmware/board/compal/highram.lds
vi osmocom-bb/src/target/firmware/board/compal/ram.lds
vi osmocom-bb/src/target/firmware/board/compal_e88/flash.lds
vi osmocom-bb/src/target/firmware/board/compal_e88/loader.lds
vi osmocom-bb/src/target/firmware/board/mediatek/ram.lds

找到里面的这一串代码            

KEEP(*(SORT(.ctors)))
在下面加入
KEEP(*(SORT(.init_array)))  

保存即可，全部修改好，在进入osmocom-bb/src重新编译一下

$ make -e CROSS_TOOL_PREFIX=arm-none-eabi-
```

# 4.实战操作

先用ttl转usb连接c118手机，插入usb孔

![](http://image.skywang.fun/picGO/20200215002734.jpg)

lsusb   //查看是否插入成功

![](http://image.skywang.fun/picGO/20200215002741.png)

```
cd /home/skywangg/osmocombb/osmocom-bb/src/host/osmocon/ 
//cd进osmocombb目录

sudo ./osmocon -m c123xor -p /dev/ttyUSB0 ../../target/firmware/board/compal_e88/layer1.compalram.bin
//键入该底下的命令
```

![](http://image.skywang.fun/picGO/20200215002748.png)

按一下开机键（短按，不是长按）

有时候停了 就按一下开机键（我有时候需要按几次才完全写入的）

![](http://image.skywang.fun/picGO/20200215002755.png)

看到这里就写入成功了。手机上也出现 layer 1 osmocom bb

![](http://image.skywang.fun/picGO/20200215002804.jpg)

**六.打开新终端输入**

新建一个终端

```
cd /home/skywangg/osmocombb/osmocom-bb/src/host/layer23/src/misc 
//cd进osmocombb目录

sudo ./cell_log -O  
//搜索附近伪基站
```

![](http://image.skywang.fun/picGO/20200215002812.png)

```
cd /home/skywangg/osmocombb/osmocom-bb/src/host/layer23/src/misc 
//cd进osmocombb下的目录

sudo ./ccch_scan -i 127.0.0.1 -a ARFCN
//刚刚搜索附近的伪基站 

sudo ./ccch_scan -i 127.0.0.1 -a 46
```

D终端
```
sudo wireshark -k -i lo   
//打开wireshark并开始，立即抓包
```
选gsm_sms即可




