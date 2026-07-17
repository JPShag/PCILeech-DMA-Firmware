# **简介**

​	CH347FPGADownloader是一款专用于CH347的FPGA下载软件，结合OpenOCD开源项目实现。

​	当前支持FPGA型号主要以xilinx为主，其中具体型号如下：

![image-20221027190604695](README.assets/image-20221027190604695.png)

# **软件使用**

## **界面显示**

![image-20221027190616419](README.assets/image-20221027190616419.png)

## **下载设置选项**

![image-20221027190632574](README.assets/image-20221027190632574.png)

​	1.“选择FPGA型号”：选择本次进行操作的FPGA型号，该选择框可编辑，可根据输入内容进行支持列表匹配；

​	2.“选择下载文件类型”：

​		A. BIT文件方式下载：此选择默认将BIT文件下载至FPGA RAM当中，且掉电丢失，上电需重新下载；

​		B. BIN文件方式下载：此选择默认将BIN文件下载至FPGA连接的FLASH当中，程序固化至FLASH，下载完毕后自动软件复位FPGA运行程序；

​	3.“选择硬件时钟”：选择CH347作为JTAG接口的TCK时钟频率，此处默认10M、15M、30M设定，若需要其他频率可手动输入；

​	4.“关于”：查看软件版本信息；

​	5.“选择下载文件”：点击根据当前下载方式来选择此次操作所需bit或bin文件，会默认记住此次选择文件路径，下次选择时进入该路径；

​	6.“下载”：BIN下载方式下将显示下载进度条，下载完毕自动隐藏，选择BIN文件下载方式时也会自定显示下载进度条；

![image-20221027190647377](README.assets/image-20221027190647377.png)

## **信息输出**

![image-20221027190701769](README.assets/image-20221027190701769.png)

​	1.“设备信息”：动态显示CH347-JTAG接口插拔信息

​	2. “清屏”：情况当前黑色显示框中内容

# **软件获取地址**

[WCHSoftGroup/ch347: ch347 480Mbps high-speed USB to Jtag/I2C/SPI/Uart/GPIO etc. (github.com)](https://github.com/WCHSoftGroup/ch347/CH347FPGADownloader)