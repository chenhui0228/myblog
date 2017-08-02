---
title: CEPH 运维之安装部署
date: 2017-08-01 14:09:41
categories:
- 分布式存储
- 运维操作
tags: 
- ceph
- 分布式存储
---

这篇文档主要介绍ceph的搭建过程。

### 集群规划 ###

服务器规划及配置，如下：

| hostname | public ip | cluster ip | 节点说明 |
| :----- | :----- | :----- | :----- |
| ch-osd-1 | 172.16.30.73 | 172.16.31.73 | osd节点 |
| ch-osd-2 | 172.16.30.74 | 172.16.31.74 | osd节点 |
| ch-osd-3 | 172.16.30.75 | 172.16.31.75 | osd节点 |
| ch-osd-4 | 172.16.30.77 | 172.16.31.77 | osd节点 |
| ch-mon-1 | 172.16.30.78 | 172.16.31.78 | mon+rgw+manger节点 |
| ch-mon-2 | 172.16.30.79 | 172.16.31.79 | mon+rgw节点 |
| ch-mon-3 | 172.16.30.80 | 172.16.31.80 | mon+rgw节点 |

- 操作系统：centos release 7.2
- CPU：OSD节点为Intel(R) Xeon(R) CPU E5-2650 v4 @ 2.20GHz \* 48 ，MON节点为Intel(R) Xeon(R) CPU E5-2650 0 @ 2.00GHz \* 32；
- 内存大小：OSD 为256GB； MON为64GB
- 数据硬盘配置（不含系统盘）：OSD 为1.2TB SAS \* 3和480G SSD \* 1，其中SSD不是必要的，我们这里主要存放journal，MON单独部署可以不需要数据盘
- 网络配置：public 网络和 cluster 均为万兆光纤
- 每台服务器第1，2块磁盘做RAID1；其余磁盘做RAID0
- ch-mon-1节点作为管理节点，部署ceph-deploy
- Ceph版本：12.1.0
- ceph-deploy版本：1.5.36
- 这里使用root用户安装，如果不是root用户，应该拥有root权限

### 环境准备 ###

#### 基础环境检查 ####

1. 网络连接正常（方法略）
1. ntp服务正常（方法略）
1. 集群服务器时间，时区一致（方法略）
1. 防火墙策略，开端口6789，6800：7300
1. SELINUX设置为Permissive或者禁掉
1. 磁盘阵列检查

首先需要在存储节点安装Megacli，下载RPM包，如MegaCli-8.07.14-1.noarch.rpm，安装命令如下：

    rpm -ivh MegaCli-8.07.14-1.noarch.rpm

使用磁盘阵列检查工具Megacli进行相关检查，查看是否满足需要的配置策略。

    /opt/MegaRAID/MegaCli64 -LDGetProp -Cache -LALL -aALL

一般情况可在安装操作系统前对各硬盘做好磁盘阵列，不同厂商的设备磁盘阵列配置略有不同，这里不做详述。如果没做磁盘阵列，这里需要做磁盘阵列，使用MegaCli来对磁盘做日常管理。

OSD数据盘做RAID0 ，并分别设置读、写、缓存策略，其中读策略为Read-Ahead；写策略为WriteBack，写入缓存直接返回；磁盘缓存Disk cache设置为disable；I/O策略为Direct请求不被系统Cache缓存。如磁盘编号为10 的物理磁盘组成一个RAID0

    /opt/MegaRAID/MegaCli64 -cfgldadd -r0 [32:10] WB RA RA -a0

或者

    /opt/MegaRAID/MegaCli64 -LDSetProp 10 WB -a0
    /opt/MegaRAID/MegaCli64 -LDSetProp 10 RA -a0
    /opt/MegaRAID/MegaCli64 -LDSetProp 10 RA -a0


SSD做日志盘做RAID0，读策略为Normal，不适用预读，写策略为Write Through直接写入磁盘；如果SSD有掉电保护，磁盘缓存Disk cache设置为Enable；I/O策略为Direct请求不被Cache缓存。

    /opt/MegaRAID/MegaCli64 -cfgldadd -r0 [32:13] WT NORA Direct -a0


Megacli更详细的使用，可以参考其他资料，如[参考文档1](https://supportforums.cisco.com/document/62901/megacli-common-commands-and-procedures)等

#### 基础环境配置 ####

这里采用yum安装，我已经将CEPH二进制RPM包上传至YUM镜像，所以先配置CEPH可用的YUM源，包括CEPH本身和EPEL源，如下：

	[root@node-5 env-check-init]# cat /etc/yum.repos.d/CentOS.repo 
    ...
    [CENTOS7-epel]
    name=CENTOS7 epel resource
    baseurl=http://yum17.int.sfdc.com.cn/epel7Server/$basearch
    enabled=1
    gpgcheck=0
    gpgkey=http://yum17.int.sfdc.com.cn/epel7/$basearch/RPM-GPG-KEY-EPEL-7
    
    [CENTOS7-ceph]
    name=CENTOS7 ceph resource
    baseurl=http://yum17.int.sfdc.com.cn/ceph/el7/$basearch
    enabled=1
    gpgcheck=0
    gpgkey=http://yum17.int.sfdc.com.cn/ceph/el7/$basearch/RPM-GPG-KEY-EPEL-7

> **提示**：如果你没有自己的YUM源可使用国内开源镜像，如[清华镜像](https://mirrors.tuna.tsinghua.edu.cn/)，[中科大镜像](http://mirrors.ustc.edu.cn/)，[阿里镜像](https://mirrors.aliyun.com/)等，也可以使用[Ceph官方文档](http://docs.ceph.com/docs/master/start/quick-start-preflight/#rhel-centos)给出的配置。

### 集群部署 ###

#### 软件包安装 ####

在管理节点安装ceph-deploy





### 常用运维 ###

#### 增加/删除 MONITORs ####

##### 增加MONITOR #####

##### 删除MONITOR #####

首先停需要删除的monitor守护进程，命令如下：

    service ceph -a stop mon.{mon-id}
	#for example
	service ceph -a stop mon.ch-mon-1

然后将monitor从集群删除

	ceph mon remove {mon-id}
	#for example
	ceph mon remove ch-mon-1

最后，删除ceph.conf中的该monitor配置

更详细的使用方法见[官方文档](http://docs.ceph.com/docs/master/rados/operations/add-or-rm-mons/#removing-monitors)


#### 增加/删除 OSDs ####

##### 增加OSD #####

##### 删除OSD #####

在删除OSD之前，OSD通常为up且in的状态，需要先将要删除的OSD踢出集群，让集群可以进行rebalancing，同时将数据拷贝到其它OSD上。 在**管理节点**执行OSD踢出集群命令如下：

    ceph osd out {osd-num}
	#for example
	ceph osd out 2

一旦OSD被踢出集群后，集群通过将要删除的OSD上的PGs迁出来进行rebalancing，我们可以通过如下命令来观察整个过程

    ceph -w

> **注意**：在只有少数服务器的小集群中，比如我们测试的集群，在进行out操作可能使一些PGs始终在active+remapped状态，这时，应该将osd置回in,回到初始状态
> `ceph osd in {osd-num}`
> 然后，不是out OSD，而是将OSD权重设置为0。
> `ceph osd crush reweight osd.{osd-num} 0`
> 之后再观察数据迁移状态。out和权重置0的区别在于，第一种情况下，OSD权重不改变，后者桶的权重被更新。

将OSD踢出集群后，OSD可能仍然处于up且out状态，在从配置中删除OSD之前，需要将该OSD服务停止，登录**OSD节点**，执行命令停止相关OSD服务

    ssh {osd-host}
    sudo systemctl stop ceph-osd@{osd-num}
	#for example
	ssh ch-osd-1
	systemctl stop ceph-osd@2

或者

	service ceph stop osd.2

或者

	kill -9 {pid}

最后需要将OSD从集群的CRUSH MAP中删除，同时删除其权限，并且从OSD MAP删除OSD，在**管理节点**执行命令：

    #remove osd from crush map
	ceph osd crush remove {name}
	#remove the osd authentication key
	ceph auth del osd.{osd-num}
	#remove the osd
	ceph osd rm {osd-num}
	#for example
	ceph osd crush remove osd.2
	ceph auth del osd.2
	ceph osd rm 2

如果在ceph.conf文件中有该OSD的配置，还需要删除相关配置

更详细的使用方法见[官方文档](http://docs.ceph.com/docs/master/rados/operations/add-or-rm-osds/#removing-osds-manual)


**原创申明**：本文为博主原创，转载请注明出处！	


