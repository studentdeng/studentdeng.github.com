---
layout: post
title: "bochs-configure"
date: 2012-02-14  23:55
comments: true
categories: [configure]
---

# Bochs + freedos安装配置

	sudo apt-get install build-essential

	sudo apt-get install xorg-dev

	sudo apt-get install libgtk2.0-dev

下载

	bochshttp://bochs.sourceforge.net/cgi-bin/topper.pl?name=See+All+Releases&url=http://sourceforge.net/projects/bochs/files

	$ tar vxaf bochs-2.5.1.tar.gz

	$ cd bochs-2.5.1

	$ ./configure –enable-debugger-enable-disasm

	$ make

	$ sudo make install

下载freedoshttp://bochs.sourceforge.net/diskimages.html 复制到工作目录下

# Bochs 配置 先通过dos引导，我们的软件复制在B盘下

	############################################################### 
	# Configuration file for Bochs 
	###############################################################

	# how much memory the emulated machine will have 
	megs: 32

	# filename of ROM images 
	romimage: file=/usr/share/bochs/BIOS-bochs-latest 
	vgaromimage: file=/usr/share/vgabios/vgabios.bin

	# what disk images will be used 
	floppya: 1_44=freedos.img, status=inserted 
	floppyb: 1_44=pm.img, status=inserted

	# choose the boot disk. 
	boot: a

	# where do we send log messages? 
	# log: bochsout.txt

	# disable the mouse 
	mouse: enabled=0

	# enable key mapping, using US layout as default. 
	keyboard_mapping: enabled=1, map=/usr/share/bochs/keymaps/x11-pc-us.map

	#enabled debug using xchg bx, bx

	magic_break:enabled=1

	可以通过下面方式来部署程序，当然，写成makefile最省事

	$ sudo mount –o loop pm.img /mnt/floppy

	$ sudo cp test.com /mnt/floppy

	$ sudo umount /mnt/floppy

# bochs 调试

调试方法很多，这里介绍最简单的方式之一。bochs 配置中增加 magic_break:enabled=1

代码中增加xchg bx, bx，bochs会停在代码出。当然，前提是bochs需要支持debug