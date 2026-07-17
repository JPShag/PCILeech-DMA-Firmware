# ch347
CH347是一款高速USB总线转接芯片，通过USB总线提供UART、I2C、SPI、JTAG等接口。

## JTAG调试/下载

本章节介绍如何在OpenOCD中添加CH347-JTAG接口，通过OpenOCD可操作CH347进行JTAG调试或下载应用。

## 为OpenOCD添加接口

OpenOCD增加CH347配置文件，用于型号识别，操作方式：

### 1、目录tcl/target下添加CH347.cfg设备文件

```
    adapter driver ch347
    ch347 vid_pid 0x1a86 0x55dd 
    adapter speed 10000
```



### 2、目录src/jtag/driver下添加ch347.c文件

​		此文件为设备驱动文件，提供JTAG core操作CH347的API接口函数

### 3、修改配置文件

**修改配置文件 configure.ac**
1、OpenOCD根目录下configure.ac文件，添加CH347接口驱动支持：

    a. m4_define([USB1_ADAPTERS]中添加CH347接口
        m4_define([USB1_ADAPTERS],
        [[[ftdi], [MPSSE mode of FTDI based devices], [FTDI]],
        [[ch347], [Mode 3 of the CH347 devices], [CH347]],
        [[stlink], [ST-Link Programmer], [HLADAPTER_STLINK]],
        [[ti_icdi], [TI ICDI JTAG Programmer], [HLADAPTER_ICDI]],
        [[ulink], [Keil ULINK JTAG Programmer], [ULINK]],
        [[usb_blaster_2], [Altera USB-Blaster II Compatible], [USB_BLASTER_2]],
        [[ft232r], [Bitbang mode of FT232R based devices], [FT232R]],
        [[vsllink], [Versaloon-Link JTAG Programmer], [VSLLINK]],
        [[xds110], [TI XDS110 Debug Probe], [XDS110]],
        [[cmsis_dap_v2], [CMSIS-DAP v2 Compliant Debugger], [CMSIS_DAP_USB]],
        [[osbdm], [OSBDM (JTAG only) Programmer], [OSBDM]],
        [[opendous], [eStick/opendous JTAG Programmer], [OPENDOUS]],
        [[armjtagew], [Olimex ARM-JTAG-EW Programmer], [ARMJTAGEW]],
        [[rlink], [Raisonance RLink JTAG Programmer], [RLINK]],
        [[usbprog], [USBProg JTAG Programmer], [USBPROG]],
        [[aice], [Andes JTAG Programmer], [AICE]]])
    
    b. AC_ARG_ENABLE 部分，添加内容如下：
        AC_ARG_ENABLE([ch347],
         AS_HELP_STRING([--enable-ch347], [Enable building support for CH347]),
         [build_ch347=$enableval], [build_ch347=no])
    
    c. AS_IF 部分，添加内容如下：
         AS_IF([test "x$build_ch347" = "xyes"], [
          AC_DEFINE([BUILD_CH347], [1], [1 if you want CH347.])
         ], [
           AC_DEFINE([BUILD_CH347], [0], [0 if you don't want CH347.])
         ])
    
    d. AM_CONDITIONAL 部分，添加内容如下：
         AM_CONDITIONAL([CH347], [test "x$build_ch347" = "xyes"])

**修改src/jtag/interfaces.c**

    a. 添加编译选项
        #if BUILD_CH347 == 1
        extern struct adapter_driver ch347_adapter_driver;
        #endif
    b. 在jtag接口列表结构体adapter_drivers中添加，此处由配置脚本来启用对应接口驱动
        #if BUILD_CH347 == 1
        	&ch347_adapter_driver,
        #endif

**修改/src/jtag/drivers/Makefile.am**

```
a. 添加编译支持
     if CH347
     DRIVERFILES += %D%/ch347_jtag.c
     endif
```

### 4、编译方法

**使用cygwin进行编译**

```
a. 执行./bootstrap
b. 执行./configure --prefix=/home/OpenOCD/CH347 --enable-ch347 --host=i686-w64-mingw32 CFLAGS='-g -o0'
c. 执行make install
d. 编译成功时/home/OpenOCD/CH347文件夹下会生成目标文件，其中bin目录下为OpenOCD可执行文件。
```

