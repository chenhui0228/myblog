---
title: CEPH运维之安装部署（luminous）
date: 2017-08-01 14:09:41
categories:
- 分布式存储
- 运维操作
- Ceph
tags: 
- ceph
- 分布式存储
---

----------

**原创申明**：本文为博主原创，转载请注明出处！	

----------

这篇文档主要介绍ceph的搭建过程。

### 集群规划 ###

服务器规划及配置，如下：

| hostname | public ip | cluster ip | 节点说明 |
| :----- | :----- | :----- | :----- |
| ch-osd-1 | 172.16.30.73 | 172.16.31.73 | osd节点 |
| ch-osd-2 | 172.16.30.72 | 172.16.31.72 | osd节点 |
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
- Ceph版本：目前最新版 v12.1.2
- ceph-deploy版本：1.5.38
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

##### yum源配置 #####

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

##### SSH互信 #####

需要为管理节点和其他集群节点建立ssh互信，使管理节点可以免验证登录其他各节点

在管理节点生成ssh keys，命令如下：

    ssh-keygen

将管理节点的ssh key 拷贝到其它个节点：

    ssh-copy-id root@172.16.30.xxx

### 集群部署 ###

#### 软件包安装 ####

在管理节点安装ceph-deploy

    yum install ceph-deploy -y

安装Ceph，在各个节点执行命令

    yum install ceph -y

安装完后可以使用如下命令查看版本

    [root@ch-mon-1 ~]# ceph -v
    ceph version 12.1.2 (b661348f156f148d764b998b65b90451f096cb27) luminous (rc)

同时，可以看到在/etc目录下新增了一个ceph目录。进入/etc/ceph目录

    cd /etc/ceph

#### 创建集群 ####


##### 初始化集群 #####

准备3个Monitor节点

	ceph-deploy new ch-mon-1 ch-mon-2 ch-mon-3

执行完毕后再该目录下可以看到有如下文件

	[root@ch-mon-1 ceph]# pwd
	/etc/ceph
	[root@ch-mon-1 ceph]# ll
	-rw-r--r-- 1 root root   805 Aug  3 13:44 ceph.conf
	-rw-r--r-- 1 root root 33736 Aug  3 13:45 ceph-deploy-ceph.log
	-rw------- 1 root root    73 Aug  3 13:43 ceph.mon.keyring

此时可以开始规划集群配置，如集群网络配置，我这里的配置如下：

默认配置

    [root@ch-mon-1 ceph]# cat ceph.conf 
    [global]
    fsid = 31fc3bef-d912-4d12-aa1e-130d3270d5db
    mon_initial_members = ch-mon-1, ch-mon-2, ch-mon-3
    mon_host = 172.16.30.78,172.16.30.79,172.16.30.80
    auth_cluster_required = cephx
    auth_service_required = cephx
    auth_client_required = cephx

修改后

    [root@ch-mon-1 ceph]# cat ceph.conf 
    [global]
    fsid = 31fc3bef-d912-4d12-aa1e-130d3270d5db
    mon_initial_members = ch-mon-1, ch-mon-2, ch-mon-3
    mon_host = 172.16.30.78,172.16.30.79,172.16.30.80
    auth_cluster_required = cephx
    auth_service_required = cephx
    auth_client_required = cephx
    
    public_network = 172.16.30.0/24
    cluster_network = 172.16.31.0/24
    osd_pool_default_size = 3
    osd_pool_default_min_size = 1
    osd_pool_default_pg_num = 8
    osd_pool_default_pgp_num = 8
    osd_crush_chooseleaf_type = 1
    
    [mon]
    mon_clock_drift_allowed = 0.5
    
    [osd]
    osd_mkfs_type = xfs
    osd_mkfs_options_xfs = -f
    filestore_max_sync_interval = 5
    filestore_min_sync_interval = 0.1
    filestore_fd_cache_size = 655350
    filestore_omap_header_cache_size = 655350
    filestore_fd_cache_random = true
    osd op threads = 8
    osd disk threads = 4
    filestore op threads = 8
    max_open_files = 655350

##### 初始化Monitor #####

部署初始的monitors，并获得keys

    ceph-deploy mon create-initial

做完这一步，在当前目录下就会看到有如下的keyrings：

	[root@ch-mon-1 ceph]# ll
	-rw------- 1 root root    71 Aug  3 13:45 ceph.bootstrap-mds.keyring
	-rw------- 1 root root    71 Aug  3 13:45 ceph.bootstrap-mgr.keyring
	-rw------- 1 root root    71 Aug  3 13:45 ceph.bootstrap-osd.keyring
	-rw------- 1 root root    71 Aug  3 13:45 ceph.bootstrap-rgw.keyring
	-rw------- 1 root root    63 Aug  3 13:45 ceph.client.admin.keyring

要在节点使用ceph命令行需要将ceph.client.admin.keyring放在需要的节点的/etc/ceph目录下。如这里希望在所有的节点使用命令行，可以通过如下命令将ceph.client.admin.keyring拷贝到各节点，当然也可以使用cp命令。

    ceph-deploy admin ch-mon-2 ch-mon-3 ch-osd-1 ch-osd-2 ch-osd-3 ch-osd-4

在L版本的Ceph中新增了manager daemon，如下命令部署一个Manager守护进程

    ceph-deploy mgr create ch-mon-1

##### 增加OSDs #####

我们使用的版本后端存储默认使用bluestore

下面我们添加OSDs

    ceph-deploy osd create ch-osd-1:/dev/sdb ch-osd-1:/dev/sdc ch-osd-1:/dev/sdd ch-osd-2:/dev/sdb ch-osd-2:/dev/sdc ch-osd-2:/dev/sdd ch-osd-3:/dev/sdb ch-osd-3:/dev/sdc ch-osd-3:/dev/sdd ch-osd-4:/dev/sdb ch-osd-4:/dev/sdc ch-osd-4:/dev/sdd

> **提示**：在早期的版本中，添加OSD分为prepare和activate两步，这里不详述

等命令执行结束之后可以查看集群状态

	[root@ch-mon-1 ~]# ceph -s
	  cluster:
	    id:     31fc3bef-d912-4d12-aa1e-130d3270d5db
	    health: HEALTH_OK
	 
	  services:
	    mon: 3 daemons, quorum ch-mon-1,ch-mon-2,ch-mon-3
	    mgr: ch-mon-1(active)
	    osd: 12 osds: 12 up, 12 in
	 
	  data:
	    pools:   0 pools, 0 pgs
	    objects: 0 objects, 0 bytes
	    usage:   12742 MB used, 13386 GB / 13398 GB avail
	    pgs:
 
查看OSDs

	[root@ch-osd-1 ~]# ceph osd tree
	ID CLASS WEIGHT   TYPE NAME         STATUS REWEIGHT PRI-AFF 
	-1       13.08472 root default                              
	-3        3.27118     host ch-osd-1                         
	 0   hdd  1.09039         osd.0         up  1.00000 1.00000 
	 1   hdd  1.09039         osd.1         up  1.00000 1.00000 
	 2   hdd  1.09039         osd.2         up  1.00000 1.00000 
	-5        3.27118     host ch-osd-2                         
	 3   hdd  1.09039         osd.3         up  1.00000 1.00000 
	 4   hdd  1.09039         osd.4         up  1.00000 1.00000 
	 5   hdd  1.09039         osd.5         up  1.00000 1.00000 
	-7        3.27118     host ch-osd-3                         
	 6   hdd  1.09039         osd.6         up  1.00000 1.00000 
	 7   hdd  1.09039         osd.7         up  1.00000 1.00000 
	 8   hdd  1.09039         osd.8         up  1.00000 1.00000 
	-9        3.27118     host ch-osd-4                         
	 9   hdd  1.09039         osd.9         up  1.00000 1.00000 
	10   hdd  1.09039         osd.10        up  1.00000 1.00000 
	11   hdd  1.09039         osd.11        up  1.00000 1.00000 

至此，整个集群就搭建完毕。

通过对目前最新版本Ceph部署，可见比老版本比如生产上大量使用的Hammer版部署起来简单，当然这里没有太多的配置优化。

### 常用运维 ###

#### 开启监控模块 ####

在配置文件/etc/ceph/ceph.conf中添加

    [mgr]
    mgr modules = dashboard

或者

    ceph mgr module enable dashboard

设置dashboard的ip和端口

    ceph config-key put mgr/dashboard/server_addr 172.16.30.78
    ceph config-key put mgr/dashboard/server_port 7000

重启mgr服务

	[root@ch-osd-1 ~]# systemctl restart ceph-mgr@ch-mon-1


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

	kill -9 {pid}

在早期的版本中使用如下命令

	service ceph stop osd.2

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

#### 配置推送 ####

我们无需再每台服务器上修改ceph.conf文件中的配置信息，只需要在管理节点修改一份，然后推送到需要更行的节点上即可，如这里我们修改了ch-mon-1上/etc/ceph/ceph.conf内容，需要同时在ch-mon-2, ch-mon-3生效

    ceph-deploy --overwrite-conf config push ch-mon-2 ch-mon-3


