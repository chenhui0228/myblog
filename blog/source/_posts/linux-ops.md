---
title: linux常用操作总结
date: 2017-08-01 14:17:19
categories:
- linux
tags:
- linux
---

----------

**原创申明**：本文为博主原创，转载请注明出处！

----------

-  查看当前文件夹大小
	
	```bash
	du -sh <dir_path>
	or
	du -ah --max-depth=1
	```

- 查看系统默认的文件句柄数

	```bash
	ulimit -n
	```

- 查看进程打开的文件句柄数】

	```bash
	lsof -n|awk '{print $2}'|sort|uniq -c|sort -nr|more
	```

- 查看某块磁盘上的读写进程

	```bash
	lsof /dev/xxx | head
	# 查看详情
	lsof -p $pid
	```

- 获取进程IO信息(内核版本>2.6.20)

	```bash
	cat /proc/$pid/io #iotop -oP
	```

- 批量删除```.pyc```文件

	```bash
	find /tmp -name "*.pyc" | xargs rm -rf
	sudo find /tmp -name "*.pyc" | xargs rm -rf
	find /tmp -name "*.pyc" -exec rm -f {}\
	```

- 使用gdb调试
	
	```bash
	gdb attach $pid
	b [module]/[function]/:[lineno]
	c #run
	n #next
	s #into func
	list #content
	...
	```

- 查看端口zhanyon

	```bash
	netstat -tpnl | grep $pid
	```

- 格式化文件系统，如xfs

	```bash
	mkfs.xfs -f /dev/sdx
	```


