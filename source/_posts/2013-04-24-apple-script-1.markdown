---
layout: post
title: "Apple Script 可执行的伪代码"
date: 2013-04-24 19:21
comments: true
categories: apple_script
---
apple script 这种脚本语言就是精炼, 可执行的伪代码

	tell application "Finder"
	
	display dialog "选择源文件目录"
	set sourcepath to quoted form of POSIX path of (choose folder)
	
	display dialog "选择输出目录"
	set destinationpath to quoted form of POSIX path of (choose folder)
	
	end tell

	do shell script "/usr/bin/php /Users/ygcurer/apigen/apigen.php --source " & sourcepath & " --destination " & destinationpath & " --title curer --charset UTF-8 --access-levels public,protected --internal no --php yes --tree yes --deprecated no --todo no --download no --source-code yes --colors yes --progressbar no --update-check no"

这里把php document generate 的shell command 包装进了application中

