---
layout: post
title: "mysql 源代码 第二天 "
date: 2013-03-21 23:48
comments: true
categories: [mysql]
---

昨天搭建好环境, 今天怀着无比兴奋的心情看一下, 一条简单的sql 背后有那些有趣的事情发生, 当然, 我们先从最简单的sql 开始.

	select * from example

当然, 对于这么庞大的项目, 我只能小小的大略一下代码结构了.

## SQL Interface

首先启动mysqld, mysql的 server 部分

mysqld.cc

	int main(int argc, char **argv) 

传说中一切世界的源头. 在处理一些参数或是配置什么的, whatever, 先不关心这个,跳转到 

	int win_main(int argc, char **argv)

还是一些配置, 读取my.ini, 日志, socket, 什么的. 然后创建一大堆的thread, 这里面大部分暂时没用
在很多的thread callback 函数中, 有一个必须要了解一下了它是

	handle_one_connection // sql_connect.cc

它是client 请求server callback的开始 然后依次调用 

	my_real_read -> do_command -> my_net_read 

这里等待socket

client 部分

通过mysql 我们连接到 mysqld, 这里我们先跳过其他命令过程. 直接步入今天的主题, 我事先已经创建好一个简单的数据库, 里面2个字段, 3条数据.

	select * from example

回到之前的server 

这时, mysqld 收到的socket, 在检查一些错误之后, 我们来到

	dispatch_command -> mysql_parse

## mysql Query Parser

这里我们进入了mysql Query Parser 部分, mysql 首先需要词法分析,语法分析 SQL 语句, 把我们的SQL 转变成一个个token 语法树什么的 要做一些编译器类似的前端部分, whatever, 对这部分不感兴趣, 我现在可没有扩展SQL 语句的想法.我们这条语句是select 所以, 我们最后跳到了 handle_select部分

	mysql_parse -> mysql_execute_command -> execute_sqlcom_select -> handle_select -> mysql_select

## Query Optimizer

这里我们来到了mysql Query Optimizer 部分, 3个主要的事情
	 prepare
	 optimize
	 exec
这个将是之后重点要学的, 现在先跳过啦

## Query Execution

	do_select -> sub_select -> sub_select ....

这里我们看到了很多类似的函数, 从名字上来看, 基本上是处理递归的, 而且,事实上也的确是这样子的....(真是废话), mysql 这里不断地读取一行行数据,
判断是否嵌套查询等等.最后 一系列的clear unlock. free, release

	net_end_statement
这里将数据返回client, server 继续
	do_command -> my_net_read ... 

最后结构图
![alt text](/images/mysql_1_1.gif)
![alt text](/images/mysql_1_2.gif)





