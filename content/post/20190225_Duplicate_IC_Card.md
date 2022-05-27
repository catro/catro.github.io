---
title: "使用PN532复制IC卡"
date: 2019-02-25T20:19:15+08:00
tags: ["PN532", "IC", "UID", "linux"]
categories: ["瞎折腾"]
draft: false
---
本文介绍复制IC卡所需要的硬件和软件，复制过程中遇到的一些问题以及解决方案。其中并不涉及IC卡存储格式或通信协议，需要了解请搜索相关资料。复制的IC卡可以置于手机及手机壳之间以达到近似于手机刷卡的效果。
<!--more-->
**硬件列表**

- [PN532模块](https://item.taobao.com/item.htm?spm=a1z09.2.0.0.257e2e8dSC4aqf&id=45375592400&_u=r1ra87b1636a): 读写IC卡，支持Mifare 1K等卡片。
- [USB转TTL](https://detail.tmall.com/item.htm?id=537486291440&spm=a1z09.2.0.0.257e2e8dSC4aqf&_u=r1ra87b1f0fe)：链接电脑和PN532模块。
- [IC/ID二合一卡](https://item.taobao.com/item.htm?spm=a1z09.2.0.0.257e2e8dSC4aqf&id=549885669295&_u=r1ra87b1db65)：IC卡可以更改UID，ID卡为T5577类型。下一篇会介绍如何复制ID卡。
- [消磁贴（可选）](https://item.taobao.com/item.htm?spm=a1z09.2.0.0.257e2e8dSC4aqf&id=549885669295&_u=r1ra87b1db65)：复制卡贴在手机后需要使用，否则读卡器无法识别。

PN532模块需要焊接4个引脚，然后将TTL线按图进行连接：
![PN532连接图](/img/PN532.jpg)

**软件列表**

- libusb: [http://libusb.sf.net](http://libusb.sf.net/)
- libnfc: [https://github.com/nfc-tools/libnfc](https://github.com/nfc-tools/libnfc)
- mfoc: [https://github.com/nfc-tools/mfoc](https://github.com/nfc-tools/mfoc)
- mfcuk: [https://github.com/nfc-tools/mfcuk](https://github.com/nfc-tools/mfcuk) 最终本人使用的是另外分支，详见复制过程。

**Linux下安装过程**

libusb: Ubuntu用户可以直接从源中安装: `sudo apt install libusb-1.0-0-dev`.
libnfc包含NFC驱动以及一些工具，根据说明文档编译安装。这里使用的是PN532模块，因此在配置的时候需要稍作修改，将对应的模块驱动编译进去。
```
./configure --with-drivers=pn532_uart --enable-serial-autoprobe --prefix=/usr --sysconfdir=/etc
make
sudo make install
```
然后是创建配置文件，假定USB转TTL的设备文件路径为`/dev/ttyUSB0`。如果Ubuntu用户找不到`/dev/ttyUSB*`，可以安装尝试从源中安装`linux-image-extra-virtual`
```
sudo mkdir /etc/nfc
sudo cp libnfc.conf.sample /etc/nfc/libnfc.conf
sudo mkdir -p /etc/nfc/devices.d
printf 'name = "PN532"\nconnstring = "pn532_uart:/dev/ttyUSB0"\n' | sudo tee /etc/nfc/devices.d/pn532.conf
```
安装完libnfc后将需要复制的IC卡放到PN532模块上，然后读取该IC卡的UID。如果无法读取到UID根据输出信息进行调整。
```
LIBNFC_LOG_LEVEL=3 nfc-list -v
```

mfoc和mfcuk是用来破解IC卡密码的两种工具，根据说明文档进行编译安装。
```
autoreconf -is
./configure
make && sudo make install
```

**复制过程**

首先用mfcuk进行密码破解：
```
mfcuk -C -R 0:A -v 2
```
这里本人遇到了个问题，软件一直报错：`mfcuk_key_recovery_block() (error code=0x03)`。经过一番搜索，发现[issue-28](https://github.com/nfc-tools/mfcuk/issues/28)里面有针对这个问题的讨论，解决这个问题的补丁被提交到了另外一个分支：[https://github.com/DrSchottky/mfcuk](https://github.com/DrSchottky/mfcuk)。所以本人又重新下载编译安装了这个分支的mfcuk。最终命令行下多了个参数`-w 6`，
```
mfcuk -C -R 0:A -v 2 -w 6
```
如果IC卡的密码较简单的话一分钟左右就可以破解出来。

然后用mfoc将整张IC卡内容读出：
```
mfoc -k ${KEY0} [-k ${KEY1}...] -O original.dmp
```
如果前面mfcuk没有成功破解密码的话也可以尝试用mfoc破解：`mfoc -O original.dmp`。

最后将IC卡内容写入空白卡：
```
nfc-mfclassic W a original.dmp original.dmp
```

**验证**

最直接的方法当然是去门禁刷卡处验证了。但是不方便的情况下可以读取复制卡的内容，然后比较复制卡和原始卡内容是否一致。

**最终效果图**

放置在手机背面，带上手机壳以后基本上感觉不到

![phone_back](/img/phone_back.jpg)
![phone_back_with_cover](/img/phone_back_with_cover.jpg)
