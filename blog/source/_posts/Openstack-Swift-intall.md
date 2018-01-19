---
title: Openstack-Swift部署指南
categories:
  - 分布式存储
  - 运维操作
  - Swift
tags:
  - openstack swift
  - 对象存储
date: 2018-01-19 15:00:55
---

----------

**原创申明**：本文为博主原创，转载请注明出处！

----------

#### 环境 ####

- 硬件

	这里只使用了一台服务器，既作为Controller Node，也作为Storage Node
	
	<table>
		<tr>
			<th width="20%">主机名</th>
			<th width="20%">IP</th>
			<th width="20%">OS</th>
			<th width="20%">磁盘</th>
			<th width="20%">文件系统</th>
		</tr>
		<tr>
			<td width="20%">sf-dev</td>
			<td width="20%">10.202.127.4</td>
			<td width="20%">Centos-7.4</td>
			<td width="20%">/dev/sdb<br/> /dev/sdc<br/> /dev/sdd</td>
			<td width="20%">XFS</td>
		</tr>
	</table>

- 软件

	我们使用[Openstack Pike](https://mirrors.tuna.tsinghua.edu.cn/centos/7/cloud/x86_64/openstack-pike/)版本

- 配置可用的Openstack源

	这里使用了清华开源镜像。配置服务器镜像：

		cd /etc/yum.repos.d/
		vim CentOS-Base.repo

	增加如下配置

		...
		[openstack]
		name=Openstack
		baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos/7/cloud/x86_64/openstack-pike/
		gpgcheck=0
		...

	使用YUM跟新库

		yum update -y


#### Swift组件安装于配置 ####

**1. 安装必要的组件包**

	# yum install openstack-swift-proxy python-swiftclient \
	  python-keystoneclient python-keystonemiddleware \
	  memcached

- 从Swift源镜像获取代理服务配置文件,并进行配置

		# curl -o /etc/swift/proxy-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/proxy-server.conf-sample?h=stable/pike

	编辑代理服务器配置文件<code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">/etc/swift/proxy-server.conf</code> 

	1. 编辑 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">[DEFAULT]</code> 段内容，配置如下内容

			[DEFAULT]
			...
			bind_port = 8080
			user = swift
			swift_dir = /etc/swift
	
	2. 编辑 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">[pipeline:main]</code>
		
			[pipeline:main]
			pipeline = catch_errors gatekeeper healthcheck proxy-logging cache container_sync bulk tempurl ratelimit tempauth copy container-quotas account-quotas slo dlo versioned_writes proxy-logging proxy-server	

	3. 编辑 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">[app:proxy-server]</code> 段内容，允许自动创建账户

			[app:proxy-server]
			use = egg:swift#proxy
			...
			account_autocreate = True

	4. 在 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">[filter:tempauth]</code> 段中，设置允许的的账户/用户

			[filter:tempauth]
			...
			user_admin_admin = admin .admin .reseller_admin
			user_test_tester = testing .admin

	5. 在 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">[filter:cache]</code> 段中，设置memcache位置

			[filter:cache]
			use = egg:swift#memcache
			...
			memcache_servers = 127.0.0.1:11211

> <font color="red">**注意：**</font>如果控制节点与存储节点分离，以上配置只需在控制节点进行配置，如果使用keystone请参考配置说明，更详细的说明请阅读[官方文档](https://docs.openstack.org/swift/latest/install/index.html)

**2. 存储节点组件包安装于配置说明**

- 安装实用软件包

		# yum install xfsprogs rsync

- 格式化磁盘

		# mkfs.xfs /dev/sdb -f
		# mkfs.xfs /dev/sdc -f
		# mkfs.xfs /dev/sdd -f

- 创建mount点

		# mkdir -p /srv/node/sdb
		# mkdir -p /srv/node/sdc
		# mkdir -p /srv/node/sdd

- 编辑 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">/etc/fstab</code> 文件，添加如下内容：

		/dev/sdb /srv/node/sdb xfs noatime,nodiratime,nobarrier,logbufs=8 0 2
		/dev/sdc /srv/node/sdc xfs noatime,nodiratime,nobarrier,logbufs=8 0 2
		/dev/sdd /srv/node/sdd xfs noatime,nodiratime,nobarrier,logbufs=8 0 2

- Mount 设备

		# mount /srv/node/sdb
		# mount /srv/node/sdc
		# mount /srv/node/sdd

- 创建并编辑 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">/etc/rsyncd.conf</code> 文件，内容如下：

		uid = swift
		gid = swift
		log file = /var/log/rsyncd.log
		pid file = /var/run/rsyncd.pid
		address = 10.202.127.4 #控制节点IP
		
		[account]
		max connections = 2
		path = /srv/node/
		read only = False
		lock file = /var/lock/account.lock
		
		[container]
		max connections = 2
		path = /srv/node/
		read only = False
		lock file = /var/lock/container.lock
		
		[object]
		max connections = 2
		path = /srv/node/
		read only = False
		lock file = /var/lock/object.lock

- 存储节点软件包安装

		# yum install openstack-swift-account openstack-swift-container \
		  openstack-swift-object

- 从Swift源镜像获取账户(accounting)、容器(container)以及对象(object)服务配置文件,并进行配置

		# curl -o /etc/swift/account-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/account-server.conf-sample?h=stable/pike
		# curl -o /etc/swift/container-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/container-server.conf-sample?h=stable/pike
		# curl -o /etc/swift/object-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/object-server.conf-sample?h=stable/pike

- 编辑账户服务配置文件 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">/etc/swift/account-server.conf</code>

	1. 在 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">[DEFAULT]</code> 中配置如下信息：

			[DEFAULT]
			...
			bind_ip = 10.202.127.4
			bind_port = 6202
			user = swift
			swift_dir = /etc/swift
			devices = /srv/node
			mount_check = True

	2. 编辑 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">[pipeline:main]</code>

			[pipeline:main]
			pipeline = healthcheck recon account-server

	3. 在 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">[filter:recon]</code> 中，设置recon cache 目录:

			[filter:recon]
			use = egg:swift#recon
			...
			recon_cache_path = /var/cache/swift

- 编辑账户服务配置文件 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">/etc/swift/container-server.conf</code>

	1. 在 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">[DEFAULT]</code> 中配置如下信息：

			[DEFAULT]
			...
			bind_ip = 10.202.127.4
			bind_port = 6201
			user = swift
			swift_dir = /etc/swift
			devices = /srv/node
			mount_check = True

	2. 编辑 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">[pipeline:main]</code>

			[pipeline:main]
			pipeline = healthcheck recon container-server

	3. 在 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">[filter:recon]</code> 中，设置recon cache 目录:

			[filter:recon]
			use = egg:swift#recon
			...
			recon_cache_path = /var/cache/swift

- 编辑账户服务配置文件 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">/etc/swift/object-server.conf</code>

	1. 在 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">[DEFAULT]</code> 中配置如下信息：

			[DEFAULT]
			...
			bind_ip = 10.202.127.4
			bind_port = 6200
			user = swift
			swift_dir = /etc/swift
			devices = /srv/node
			mount_check = True

	2. 编辑 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">[pipeline:main]</code>

			[pipeline:main]
			pipeline = healthcheck recon object-server

	3. 在 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">[filter:recon]</code> 中，设置recon cache 目录:

			[filter:recon]
			use = egg:swift#recon
			...
			recon_cache_path = /var/cache/swift
			recon_lock_path = /var/lock

- Mount 点目录所属权限设置

		# chown -R swift:swift /srv/node

- 创建 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">recon</code>目录，并设置目录所属权限

		# mkdir -p /var/cache/swift
		# chown -R root:swift /var/cache/swift
		# chmod -R 775 /var/cache/swift

**3. 创建Rings**

在启动对象存储服务前，需要创建并初始化account，container，object的rings。这里使用一个region，1个zone。总共设置2^10（1024）个最大分区（partitions），使用3副本策略（replicas），设置1小时来限制需要多次移动分区时为最小间隔

- 创建account ring

		cd /etc/swift
		# swift-ring-builder account.builder create 10 3 1

	添加所有存储到ring

		# swift-ring-builder account.builder add \
		  --region 1 --zone 1 --ip 10.202.127.4 --port 6202 --device sdb --weight 100
		# swift-ring-builder account.builder add \
		  --region 1 --zone 1 --ip 10.202.127.4 --port 6202 --device sdc --weight 100
		# swift-ring-builder account.builder add \
		  --region 1 --zone 1 --ip 10.202.127.4 --port 6202 --device sdd --weight 100

	Rebalance the ring

		# swift-ring-builder account.builder rebalance

	确认ring

		# swift-ring-builder account.builder

- 创建container ring

		cd /etc/swift
		# swift-ring-builder container.builder create 10 3 1

	添加所有存储到ring

		# swift-ring-builder container.builder add \
		  --region 1 --zone 1 --ip 10.202.127.4 --port 6201 --device sdb --weight 100
		# swift-ring-builder container.builder add \
		  --region 1 --zone 1 --ip 10.202.127.4 --port 6201 --device sdc --weight 100
		# swift-ring-builder container.builder add \
		  --region 1 --zone 1 --ip 10.202.127.4 --port 6201 --device sdd --weight 100

	Rebalance the ring

		# swift-ring-builder container.builder rebalance

	确认ring

		# swift-ring-builder container.builder

- 创建object ring

		cd /etc/swift
		# swift-ring-builder object.builder create 10 3 1

	添加所有存储到ring

		# swift-ring-builder object.builder add \
		  --region 1 --zone 1 --ip 10.202.127.4 --port 6200 --device sdb --weight 100
		# swift-ring-builder object.builder add \
		  --region 1 --zone 1 --ip 10.202.127.4 --port 6200 --device sdc --weight 100
		# swift-ring-builder object.builder add \
		  --region 1 --zone 1 --ip 10.202.127.4 --port 6200 --device sdd --weight 100

	Rebalance the ring

		# swift-ring-builder object.builder rebalance

	确认ring

		# swift-ring-builder object.builder

> <font color="red">**注意：**</font>如果有多个存储节点，需要将ring配置文件 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">account.ring.gz</code>, <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">container.ring.gz</code>，<code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">object.ring.gz</code> 分发到各个存储节点的 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">/etc/swift</code> 目录下

**3. 配置swift.conf 并启动各组件服务**

- 从swift git仓库中获取配置文件到 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">/etc/swift/</code> 目录

		# curl -o /etc/swift/swift.conf \
		  https://git.openstack.org/cgit/openstack/swift/plain/etc/swift.conf-sample?h=stable/pike
	
	编辑配置文件<code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">/etc/swift/swift.conf</code>

	1. 在 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">[swift-hash]</code> 中，设置如下内容：

			[swift-hash]
			...
			swift_hash_path_suffix = HASH_PATH_SUFFIX #替换为自己的内容，如swift
			swift_hash_path_prefix = HASH_PATH_PREFIX #替换为自己的内容，如swift

	2. 创建并配置默认策略

			[storage-policy:0]
			...
			name = Policy-0
			default = yes

	3. 如果是多存储节点，将配置文件 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">swift.conf</code> 拷贝到各节点的 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">/etc/swift</code> 目录下

	4. 修改所有节点 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">/etc/swift/</code> 目录所属权限如下：

			# chown -R root:swift /etc/swift

- 控制节点启动代理服务和memcache服务

		# systemctl enable openstack-swift-proxy.service memcached.service
		# systemctl start openstack-swift-proxy.service memcached.service

- 存储节点启动如下服务

		# systemctl enable openstack-swift-account.service openstack-swift-account-auditor.service \
		  openstack-swift-account-reaper.service openstack-swift-account-replicator.service
		# systemctl start openstack-swift-account.service openstack-swift-account-auditor.service \
		  openstack-swift-account-reaper.service openstack-swift-account-replicator.service
		# systemctl enable openstack-swift-container.service \
		  openstack-swift-container-auditor.service openstack-swift-container-replicator.service \
		  openstack-swift-container-updater.service
		# systemctl start openstack-swift-container.service \
		  openstack-swift-container-auditor.service openstack-swift-container-replicator.service \
		  openstack-swift-container-updater.service
		# systemctl enable openstack-swift-object.service openstack-swift-object-auditor.service \
		  openstack-swift-object-replicator.service openstack-swift-object-updater.service
		# systemctl start openstack-swift-object.service openstack-swift-object-auditor.service \
		  openstack-swift-object-replicator.service openstack-swift-object-updater.service

#### 服务验证 ####

新建并编辑 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">testrc</code> 文件

	cd $HOME
	vim testrc

写入 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">/etc/swift/proxy-server.conf</code> 配置文件中 配置的用户权限信息，如下：

	export ST_AUTH=http://10.202.127.4:8080/auth/v1.0
	export ST_USER=test:tester
	export ST_KEY=testing

执行上诉配置脚本，使配置信息生效

	. testrc

查看swift服务状态

	swift stat

获取 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">X-Storage-Url</code> 和 <code style="color:#e74c3c; padding:1px 3px; border: solid 1px #e1e4e5">X-Auth-Token</code>：

	curl -v -H 'X-Storage-User: test:tester' -H 'X-Storage-Pass: testing' http://127.0.0.1:8080/auth/v1.0

查看账户

	curl -v -H 'X-Auth-Token: <token-from-x-auth-token-above>' <url-from-x-storage-url-above>

我们可以使用如下命令获取账户下容器列表

	swift list

获取容器下的对象列表

	swift list <container>





