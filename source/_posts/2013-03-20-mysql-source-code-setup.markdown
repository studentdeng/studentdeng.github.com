---
layout: post
title: "调试 mysql源代码 环境搭建"
date: 2013-03-20 22:06
comments: true
categories: [mysql]
---

下载源代码 [mysql](http://dev.mysql.com/downloads/mysql/)

下载CMAKE [download](http://www.cmake.org/)

代码根目录 执行下面代码, 确定生成configure.data 文件
	
	wscript win\configure.js WITH_INNOBASE_STORAGE_ENGINE WITH_PARTITION_STORAGE_ENGINE MYSQL_SERVER_SUFFIX=-pro

找到 win 目录下的 XXX.bat 文件复制到源代码根目录下, 我这里使用vs2008 ,所以复制 build-vs9.bat 到源代码根目录
在系统根目录 C:/windows 下 增加一个 my.ini 用于 mysql 配置, 我的mysql 源代码目录在 C:/mysql_code/mysql-5-1.1.68

	[mysqld]
	# set basedir to your installation path
	basedir=C:/mysql_code/mysql-5-1.1.68
	# set datadir to the location of your data directory
	datadir=C:/mysql_code/mysql-5-1.1.68/win/data

将sql目录下的share文件夹复制到源代码根目录下

修改sql_locale.cc 文件, 把文件编码改成 UTF8 with BOM, windows 你懂的

然后可以调试了, 不过这里我遇到一个问题, 在 sql_locale.cc 函数 test_lc_time_sz 函数, 这里DEBUG_ASSERT(0)了一下, 目前先暂时注释掉它 : (
现在可以跑
	mysqld
然后在
	mysql -u root -p 
over!
	
