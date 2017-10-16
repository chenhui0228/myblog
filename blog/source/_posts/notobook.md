---
title: 笔记
categories:
  - 笔记
tags:
  - 操作命令
date: 2017-09-11 14:03:07
---

----------

**原创申明**：本文为博主原创，转载请注明出处！

----------

#### pip安装理想python包 ####

##### 准备python包 #####

在一台可以连网得服务器上，编写requirements文件，或者根据已经安装好的包生成requirements文件。

    pip list #查看安装的包
    pip freeze >requirements.txt
    pip download -d /data/packages -r requirements.txt

##### 离线情况安装打包好的包 #####

将packages文件夹和requirement.txt拷贝至离线机器上目录下

    pip install --no-index --find-index=/data/packages -r requirements.txt
