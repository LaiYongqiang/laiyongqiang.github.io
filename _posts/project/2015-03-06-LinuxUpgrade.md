---
layout: post
title: 基于RAMDISK文件系统的LINUX在线升级方案
category: project
description: 基于RAMDISK文件系统的LINUX在线升级方案
---
# 基于RAMDISK文件系统的LINUX在线升级方案

标签（空格分隔）： 科技 LINUX 升级 RAMDISK 文件系统

---
## 系统概述
--- 
TSIP+子板采用`Xilnx`可编程逻辑器件解决方案，利用嵌入式ARM核运行Linux操作系统。由于子板本身无需保存任何配置，子板的所有配置均由子板下发，故子板采用RAMDISK文件系统。

--- 

## 系统分区
---
根据系统的具体情况，本系统分为以下分区：
>* uboot分区：包含系统启动程序`fsbl.elf`、逻辑文件`logic.bit`以及uboot程序`uboot.elf`
>* Device Tree分区：设备树文件
>* APP0分区：用户程序第一分区
>* Kernel分区：Linux内核映像分区
>* FileSystem分区：RAMDISK文件系统分区
>* APP1分区：用户程序第二分区

具体分区为：

|分区名字|起始地址|长度|
|:---:|:---:|:---:|
|Uboot|0x0|0x00600000|
|DeviceTree|0x00600000|0x00020000|
|APP0|0x00620000|0x007E0000|
|Kernel|0x00E00000|0X00500000|
|FileSystem|0x01300000|0x02000000|
|APP1|0X03300000|0x00D00000|

在线升级时，我们需要将相应的文件烧录到对应的文件分区去，所以在文件分区固定后，在升级时，如果烧入的文件大于目的分区，那样将无法顺利升级，故在设计系统分区时需要注意一下几点：
1. 每个分区注意留下一定的空余空间以备以后使用
2. 大小比较稳定的分区尽量靠前，比如Uboot分区、DeviceTree分区以及Kernel分区，未来大小变化较大的分区靠后，比如RAMDISK分区、用户程序APP分区等
3. 用户程序分区可以分为多个分区，在不同的应用场景下选择性加载其中几个分区

---
## 启动流程
---
系统上电后，`fsbl.elf`首先被执行，执行后加载逻辑文件`Logic.bit`,加载完成后执行`Uboot.elf`启动uboot加载`Kernel`、`FileSystem`，系统起来后执行`FileSystem`里面的`Rc.s`,在`Rc.s`中将会运行我们自定义的一个APP分区加载程序`nboot.elf`,在`nboot.elf`，它将先把Flash中APP分区的程序加载进来，然后执行里面的*.sh文件，进而调用应用程序，而在应用程序中，我们还可以根据nboot的原理实时根据需要加载不同恭喜库、组件实现不同的功能以及工作模式。有关Uboot、FileSystem、nboot的相关代码将在下面一一描述：
### Uboot
在Uboot中如果要实现以上的系统分区主要要修改`/u-boot-xlnx/include/configs/zynq_common.h`让uboot知晓当前系统的系统分区，Kernel、DeviceTree、FileSystem的起始位置以便在加载引导内核的时候传递正确的参数给Kernel，具体修改为：
```
#define CONFIG_EXTRA_ENV_SETTINGS	\
	"ethaddr=00:0a:35:00:01:22\0"	\
	"kernel_image=uImage\0"	\
	"ramdisk_image=uramdisk.image.gz\0"	\
	"devicetree_image=devicetree.dtb\0"	\
	"bitstream_image=system.bit.bin\0"	\
	"boot_image=BOOT.bin\0"	\
	"loadbit_addr=0x100000\0"	\
	"loadbootenv_addr=0x2000000\0" \
	"kernel_size=0x500000\0"	\
	"devicetree_size=0x20000\0"	\
	"ramdisk_size=0x02000000\0"	\
	"boot_size=0xF00000\0"	\
	"fdt_high=0x1B000000\0"	\
```
...
```
	"nandboot=echo Copying Linux from NAND flash to RAM... && " \
		"echo Copying Kernel.. && " \
		"nand read 0x3000000 0xE00000 ${kernel_size} && " \
		"echo Copying device tree.. && " \
		"nand read 0x2A00000 0x600000 ${devicetree_size} && " \
		"echo Copying ramdisk... && " \
		"nand read 0x6000000 0x01300000 ${ramdisk_size} && " \
		"bootm 0x3000000 0x6000000 0x2A00000\0" \
```

### 应用程序加载程序nboot
在Uboot顺利加载内核启动后会通过文件系统中的rc.s文件初始化系统，然后调用应用程序加载程序nboot。nboot启动后主要完成以下时期：
1.从Flash分区中读取压缩文件
2.解压压缩文件
3.执行压缩文件中的*.sh

相关源代码如下：
```
#include <stdio.h>
#include <mtd/mtd-user.h>
#include <string.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>

#define  TSIP_SOFTWARE_BUF_LEN  ( 15 * 1024 * 1024 )
#define  COMPRESS_HEADER_LENGTH		   4

int main()
{
	printf("COMPILE:%s  %sc\n", __DATE__, __TIME__);
    printf("##########Wait for running TSIP+ software##########\n");
    FILE* flash_bin;
    FILE* dest_file;

    //创建文件夹
    system("mkdir /tmp/APP");
    system("chmod 777 /tmp/APP");

	//创建文件
    unsigned int file_size = 0;
    flash_bin = fopen("/dev/mtd2","rb+");
    if ( flash_bin == NULL )
    {
    	printf("[%s] create fail !!!!","/dev/mtd5");
    }

    dest_file = fopen("/tmp/APP/app.tar.bz2","wb+");
    if ( dest_file == NULL )
    {
    	printf("[%s] create fail !!!!","/tmp/APP/app.tar.bz2");
    }

    //接收App压缩包，并复制到相应目录
    fseek(flash_bin,0,SEEK_END);
    file_size = ftell(flash_bin);
    fseek(flash_bin,0,SEEK_SET);
	unsigned char *appTempBuffer = (unsigned char *)malloc(TSIP_SOFTWARE_BUF_LEN);
    fread(appTempBuffer, 1, file_size, flash_bin);
    fwrite(appTempBuffer,1, file_size, dest_file);
	fclose(dest_file);
    fclose(flash_bin);
    free(appTempBuffer);

	//解压APP包，由APP包里边的shell脚本运行相应的程序
    //解压APP包，由APP包里边的shell脚本运行相应的程序

	chdir("/tmp/APP");
	system("tar xjvf /tmp/APP/app.tar.bz2");
	system("chmod 777 *.sh");
	system("./*.sh");
	system("cd /");

    return 0;
}
```
---
## 在线升级
---
在现有的文件分区的基础下，如果要对其进行在线升级需要解决的主要是一下问题：
1. 子板接收到新的子板软件
2. 将新的子板软件正确烧录到对应的分区

###接收新的子板软件
对于第一个问题，在制作升级文件时，我们可以把需要更新的升级文件以及对于的批处理文件*.sh打包成一个文件然后制作成升级文件，然后通过tftp或其他方式发送给子板，在本系统中，主板将子板软件打包成一个一个TS包，然后发送给子板，子板接收后TS包后将其保存到一块临时分配的内存中，全部收完之后再将其整体写入到一个文件内，然后解压这个文件，这样子板就接收到了新的子板软件。
相关源代码如下：
```
if (D7ProSoftware == NULL)
{
    D7ProSoftware = (U8 *) malloc( D7Pro_SOFTWARE_BUF_LEN );
    if (D7ProSoftware == NULL)
    {
        comm_debug("[COMM_ParseData] malloc fail, upgrade firmware fail\r\n");
        Comm_Cmd(COMM_SUBCMD_DOWNSOFTWARE_ERR,u32SerialNum);
        break;
    }

    comm_debug("[COMM_ParseData] malloc succ, begin to download firmware\r\n");

    pFirmWarePos = D7ProSoftware;
    u32D7ProUpdateStatus = 0;
    uActualFirmwareLen = 0;
    g_u32Upgrade_Flag = TRUE;
}

if(u16PacketSerialNum == u16LocalPacketNum)
{ 
    memcpy(pFirmWarePos, (void *)&SubBoard_ReceiveBuf[Index][20],u8Length - 4);
    pFirmWarePos += ( u8Length - 4 );
    uActualFirmwareLen += ( u8Length - 4 );
    u16LocalPacketNum++;    
    u32Receive_LastPacketSerialNum = u16PacketSerialNum;
    
    if(u16PacketSerialNum == u32PacketNum)
    {
        //int number;
        printf("[COMM_ParseData]Download file success, length=%d\r\n", uActualFirmwareLen);

        src_fd = fopen("/tmp/update.bz2","wb+");
        if( NULL== src_fd )
        {
            printf("Unable to open file: \"%s\"\n","/update.bz2");
        }
        
        filesize = fwrite(D7ProSoftware,1,uActualFirmwareLen,src_fd);                    
        if(filesize == 0)
        {
            printf("write flash err!\n");
        }

        // umcompress update files and run it's shell script
        fclose(src_fd);
        system("mkdir /tmp/extract");
        chdir("/tmp/extract");
        system("tar -xjvf /tmp/update.bz2 -C /tmp/extract");
        chdir("/tmp/extract/update");
        system("ls -l");
        system("chmod 777 /tmp/extract -R");
        system("./*.sh");
        system("echo updata!");
        system("rm /tmp/extract -fr");
        chdir("/");

        printf("Down TSIP+ firmware finish, start TS upgrade!\r\n");
        free(D7ProSoftware);
        D7ProSoftware = NULL;
        pFirmWarePos = D7ProSoftware;
        uActualFirmwareLen = 0;
        u32D7ProUpdateStatus = 1;
        u16LocalPacketNum = 1;
        g_u32Upgrade_Flag = FALSE;
    }
}
```

###烧录新的子板软件
从以上所述，在子板接收新的软件的同时还接收了一个相应的批处理文件*.sh，而且子板在解压完升级文件包之后执行了这个批处理文件，所以，我们是通过这个批处理文件完成了将新的子板软件烧录到子板对应分区的工作，批处理文件的内容为：
```
echo "flash u-boot"
flash_eraseall /dev/mtd0
nandwrite -p /dev/mtd0 u-boot.bin

echo "flash Dtb" 
flash_eraseall /dev/mtd1
nandwrite -p /dev/mtd1 zynq-zed.dtb

echo "flash APP0"
flash_eraseall /dev/mtd2
nandwrite -p /dev/mtd2 app.bin

echo "flash Kernel"
flash_eraseall /dev/mtd3
nandwrite -p /dev/mtd3 uImage

echo "flash rootfs"
flash_eraseall /dev/mtd4
nandwrite -p /dev/mtd4 uramdisk.image.gz


echo "flash ffmpeg"
flash_eraseall /dev/mtd5
nandwrite -p /dev/mtd5 ffmpeg2.2.tar.bz2
```
在这个地方我们需要注意到是在烧录新的子板软件到对应的flash分区时，我们有两个方案可选，第一个是通过`flashcp`指令直接copy新的子版软件到对应的flash分区，另外一个是先通过`flash_eraseall`命令擦除整个分区，然后通过`nandwrite`命令将新的子板软件写到对应的分区。
对于以上两个两种方案，我们可以根据网上资料[擦除flash flash_erase && flash_eraseall ](http://blog.chinaunix.net/uid-26611973-id-3380558.html)对他们的用法做进一步的了解：
>* 命令：flashcp
作用:copy　数据到 flash 中
用法：
usage: flashcp [ -v | --verbose ] <filename> <device>
flashcp -h | –help
filename:待写入的数据
device: 写入的分区，如/dev/mtd0
eg:
filename制作：mkfs.jffs2 -e 0×20000 -d cq8401 -o cq8401.img  -n  //这里的-e 0×20000 必须更你芯片的erasesize 相等
./flashcp cq8401.img /dev/mtd0  // copy cq8401.img文件系统到  /dev/mtd0分区中
当然这个命令的功能跟 dd if=/tmp/fs.img of=/dev/mtd0差不多

>* 命令：flash_eraseall
作用：擦出整个分区的数据,同时也会作**坏块检测**
用法：
flash_eraseall [OPTION] MTD_DEVICE
-q, –quiet    不显示打印信息
-j, –jffs2    一jffs2 格式化分区
eg: ./flash_eraseall -j /dev/mtd0

>* 命令：nandwrite
作用：向nand flash中写数据
用法：
nandwrite [OPTION] MTD_DEVICE INPUTFILE
-a, –autoplace       Use auto oob layout
-j, –jffs2           force jffs2 oob layout (legacy support)
-y, –yaffs           force yaffs oob layout (legacy support)
-f, –forcelegacy     force legacy support on autoplacement enabled mtd device
-n, –noecc           write without ecc
-o, –oob             image contains oob data
-s addr, –start=addr set start address (default is 0)
-p, –pad             pad to page size
-b, –blockalign=1|2|4 set multiple of eraseblocks to align to
-q, –quiet           don’t display progress messages
–help            display this help and exit
–version         output version information and exit
eg: ./nandwrite -p /dev/mtd0  /tmp/rootfs.jffs2

在使用flashcp指令时，它也会先将对应的分区擦除，然后再将新的文件写入到这个分区。
但是在flash有**坏块**的时候，flashcp擦除过程中遇到坏块会终止当前操作，这样在进行在线升级操作的时候这个分区就相当于只是被擦除了，如果坏块在uboot分区，那样重启后整个系统将无法启动。
在这种情况下，如果我们使用`flash_eraseall`+`nandwrite`,这样在使用`flash_eraseall`去擦除对应分区的时候，它会自动对其进行坏块检测，记录下当前分区的坏块，然后再通过`nandwrite`将新的软件写入到当前分区，这样就能够很好地处理在flash有坏块的情况下的烧录。(更多信息可参考[flashcp failed (bad block)?](http://lists.infradead.org/pipermail/linux-mtd/2010-March/029312.html))
所以在本系统中采取第二种方案将软件烧录到对应分区。

##总结
本系统采用这种升级方案后能够很方便快捷对软件进行升级，通过批处理文件进行部分操作能够大大扩展系统的功能,在后面可以通过这个机制完成不同应用程序启动、启动前环境变量检查、启动模式、升级条件检查、升级方式等功能，提高产品竞争力。
