# Glusterfs
## Glusterfs概述
## Glusterfs安装
```
官方地址：https://www.gluster.org/

1. 安装gluterfs源
安装epel源
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo

查看有哪些版本的镜像
[root@glusterfs1 ~]# yum search  centos-release-gluster   
=================================== N/S matched: centos-release-gluster ===================================
centos-release-gluster-legacy.noarch : Disable unmaintained Gluster repositories from the CentOS Storage
                                     : SIG
centos-release-gluster40.x86_64 : Gluster 4.0 (Short Term Stable) packages from the CentOS Storage SIG
                                : repository
centos-release-gluster41.noarch : Gluster 4.1 (Long Term Stable) packages from the CentOS Storage SIG
                                : repository
centos-release-gluster5.noarch : Gluster 5 packages from the CentOS Storage SIG repository
centos-release-gluster6.noarch : Gluster 6 packages from the CentOS Storage SIG repository

安装gluster6的源
[root@glusterfs1 ~]# yum install centos-release-gluster6.noarch

2. 安装glusterfs-server
[root@glusterfs1 ~]# yum install glusterfs-server -y

3. 启动glusterfs
[root@glusterfs1 ~]# systemctl start glusterd
[root@glusterfs1 ~]# systemctl enable glusterd
Created symlink from /etc/systemd/system/multi-user.target.wants/glusterd.service to /usr/lib/systemd/system/glusterd.service.

4. 将服务器加入存储池中
[root@glusterfs1 ~]# gluster peer probe glusterfs2
peer probe: success. 
[root@glusterfs1 ~]# gluster peer probe glusterfs3
peer probe: success.

[root@glusterfs2 ~]# gluster peer status
Number of Peers: 2

Hostname: glusterfs1
Uuid: c2ee8d16-04c7-48d9-a0cb-f6bdd3f96b2c
State: Peer in Cluster (Connected)

Hostname: glusterfs3
Uuid: a500fa96-c84a-42f7-802f-3b9b29887f3d
State: Peer in Cluster (Connected)
```
## glusterfs配置前的准备工作
```
格式化分区并挂载
[root@glusterfs2 ~]# mkfs.xfs /dev/glusterfs/data01
mkdir /data01meta-data=/dev/glusterfs/data01  isize=512    agcount=4, agsize=655104 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2620416, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@glusterfs2 ~]# mkdir /data01
[root@glusterfs2 ~]# echo "/dev/glusterfs/data01 /data01 xfs defaults 0 0" >> /etc/fstab
[root@glusterfs2 ~]# mount -a
```
## 创建volume
### GlusterFS 五种卷　　
- Distributed：分布式卷，文件通过 hash 算法随机分布到由 bricks 组成的卷上。
- Replicated: 复制式卷，类似 RAID 1，replica 数必须等于 volume 中 brick 所包含的存储服务器数，可用性高。
- Striped: 条带式卷[已弃用]，类似 RAID 0，stripe 数必须等于 volume 中 brick 所包含的存储服务器数，文件被分成数据块，以 Round Robin 的方式存储在 bricks 中，并发粒度是数据块，大文件性能好。
- Distributed Striped[已弃用]: 分布式的条带卷，volume中 brick 所包含的存储服务器数必须是 stripe 的倍数（>=2倍），兼顾分布式和条带式的功能。
- Distributed Replicated: 分布式的复制卷，volume 中 brick 所包含的存储服务器数必须是 replica 的倍数（>=2倍），兼顾分布式和复制式的功能。
- Dispersed - 分散卷，基于擦除代码，提供节省空间的磁盘或服务器故障保护。它将原始文件的编码片段存储到每个块中，其方式是仅需要片段的子集来恢复原始文件。管理员在创建卷时配置可丢失而不会丢失数据访问权限的砖块数。
- Distributed Dispersed  - 分布式分散卷，在分散的子卷中分发文件。这与分发复制卷具有相同的优点，但使用分散将数据存储到块中。

> 分布式复制卷的brick顺序决定了文件分布的位置，一般来说，先是两个brick形成一个复制关系，然后两个复制关系形成分布。
> 企业一般用后两种，大部分会用分布式复制（可用容量为 总容量/复制份数），通过网络传输的话最好用万兆交换机，万兆网卡来做。这样就会优化一部分性能。它们的数据都是通过网络来传输的。

### 分布式卷
![分布式卷](https://cloud.githubusercontent.com/assets/10970993/7412364/ac0a300c-ef5f-11e4-8599-e7d06de1165c.png)
```
1. 创建分布式卷
[root@glusterfs1 ~]# gluster volume create gv1 glusterfs1:/data01 glusterfs2:/data01 force
volume create: gv1: success: please start the volume to access data

2. 启动卷gv1
[root@glusterfs1 ~]# gluster volume start gv1
volume start: gv1: success

3. 查看gv1卷信息
[root@glusterfs3 ~]# gluster volume info 
Volume Name: gv1
Type: Distribute     # 分布式卷
Volume ID: c6bdbac2-eb32-4b75-9cdb-ee5cba8f6a98
Status: Started
Snapshot Count: 0
Number of Bricks: 2
Transport-type: tcp
Bricks:
Brick1: glusterfs1:/data01
Brick2: glusterfs2:/data02
Options Reconfigured:
transport.address-family: inet
nfs.disable: on

4. 挂载目录
[root@glusterfs3 ~]# mount -t glusterfs 127.0.0.1:/gv1 /mnt     # 挂载目录
[root@glusterfs3 ~]# df -h
Filesystem                    Size  Used Avail Use% Mounted on
/dev/mapper/jcxvg-root         20G  1.7G   18G   9% /
/dev/sda2                     497M  123M  375M  25% /boot
/dev/mapper/glusterfs-data01   10G   33M   10G   1% /data01
tmpfs                          98M     0   98M   0% /run/user/0
127.0.0.1:/gv1                 20G  270M   20G   2% /mnt        # 卷大小为两台机器盘的和
```
### 复制卷
![复制卷](https://cloud.githubusercontent.com/assets/10970993/7412379/d75272a6-ef5f-11e4-869a-c355e8505747.png)
```
1. 创建复制卷
[root@glusterfs1 ~]# gluster volume create gv1 replica 2 glusterfs1:/data01 glusterfs2:/data01 force
volume create: gv1: success: please start the volume to access data

2. 启动卷
[root@glusterfs1 ~]# gluster volume start gv1
volume start: gv1: success

3. 查看gv1卷信息
[root@glusterfs1 ~]# gluster volume info
Volume Name: gv1
Type: Replicate         # 复制卷
Volume ID: a185549a-620b-4930-b3ca-b8ad170fc3ed
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x 2 = 2
Transport-type: tcp
Bricks:
Brick1: glusterfs1:/data01
Brick2: glusterfs2:/data01
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
performance.client-io-threads: off

4. 挂载目录
[root@glusterfs1 ~]# mount -t glusterfs 127.0.0.1:/gv1 /mnt
[root@glusterfs1 ~]# df -h
Filesystem                    Size  Used Avail Use% Mounted on
/dev/mapper/jcxvg-root         20G  2.0G   18G  10% /
/dev/mapper/glusterfs-data01   10G   33M   10G   1% /data01
tmpfs                          98M     0   98M   0% /run/user/0
127.0.0.1:/gv1                 10G  135M  9.9G   2% /mnt        # 容量为1/2，相当于raid1
```
### 分布式复制卷
![分布式复制卷](https://cloud.githubusercontent.com/assets/10970993/7412402/23a17eae-ef60-11e4-8813-a40a2384c5c2.png)
```
1. 创建复制卷
[root@glusterfs1 ~]# gluster volume create gv1 replica 2 glusterfs1:/data01 glusterfs2:/data01 force        # 也可以一次性创建
volume create: gv1: success: please start the volume to access data

2. 加入两块卷
[root@glusterfs1 ~]# gluster volume add-brick gv1 replica 2 glusterfs1:/data02 glusterfs2:/data02 force
volume add-brick: success

3. 查看gv1信息
[root@glusterfs1 ~]# gluster volume info
Volume Name: gv1
Type: Distributed-Replicate     # 分布式复制卷
Volume ID: f7349958-3622-4681-ac22-727b1d6f2aad
Status: Created
Snapshot Count: 0
Number of Bricks: 2 x 2 = 4
Transport-type: tcp
Bricks:
Brick1: glusterfs1:/data01
Brick2: glusterfs2:/data01
Brick3: glusterfs1:/data02
Brick4: glusterfs2:/data02
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
performance.client-io-threads: off
```
### 分散卷
```
1. 创建分散卷
[root@glusterfs1 ~]# gluster volume create gv1 disperse 4 glusterfs1:/data01 glusterfs2:/data01 glusterfs1:/data02 glusterfs2:/data02 force
There isn't an optimal redundancy value for this configuration. Do you want to create the volume with redundancy 1 ? (y/n) y
volume create: gv1: success: please start the volume to access data

2. 启动分散卷
[root@glusterfs1 ~]# gluster volume start gv1
volume start: gv1: success

3. 查看gv1信息
[root@glusterfs1 ~]# gluster volume info
Volume Name: gv1
Type: Disperse      # 分散卷
Volume ID: 6e64eddf-6dcb-4cb8-9fa4-dd31db88f94f
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x (3 + 1) = 4
Transport-type: tcp
Bricks:
Brick1: glusterfs1:/data01
Brick2: glusterfs2:/data01
Brick3: glusterfs1:/data02
Brick4: glusterfs2:/data02
Options Reconfigured:
transport.address-family: inet
nfs.disable: on

4. 挂载分区
[root@glusterfs1 ~]# mount -t glusterfs 127.0.0.1:/gv1 /mnt/
[root@glusterfs1 ~]# df -h
Filesystem                    Size  Used Avail Use% Mounted on
/dev/mapper/jcxvg-root         20G  2.0G   18G  10% /
/dev/mapper/glusterfs-data02  5.0G   33M  5.0G   1% /data02
/dev/mapper/glusterfs-data01  5.0G   33M  5.0G   1% /data01
127.0.0.1:/gv1                 15G  250M   15G   2% /mnt        # 容量为3/4，相当于raid5
```
### 分布式分散卷
```
1. 创建分布式分散卷
[root@glusterfs1 ~]# gluster volume create gv1 disperse 3 glusterfs1:/data01 glusterfs2:/data01 glusterfs3:/data01 glusterfs1:/data02 glusterfs2:/data02 glusterfs3:/data02 force
volume create: gv1: success: please start the volume to access data

2. 启动分布式分散卷
[root@glusterfs1 ~]# gluster volume start gv1
volume start: gv1: success

3. 查看gv1信息
[root@glusterfs1 ~]# gluster volume info 
Volume Name: gv1
Type: Distributed-Disperse
Volume ID: 89667be1-29fa-4362-a704-aed85e238345
Status: Created
Snapshot Count: 0
Number of Bricks: 2 x (2 + 1) = 6
Transport-type: tcp
Bricks:
Brick1: glusterfs1:/data01
Brick2: glusterfs2:/data01
Brick3: glusterfs3:/data01
Brick4: glusterfs1:/data02
Brick5: glusterfs2:/data02
Brick6: glusterfs3:/data02
Options Reconfigured:
transport.address-family: inet
nfs.disable: on

4. 挂载分区
[root@glusterfs1 ~]# mount -t glusterfs 127.0.0.1:/gv1 /mnt/
[root@glusterfs1 ~]# df -h
Filesystem                    Size  Used Avail Use% Mounted on
/dev/mapper/jcxvg-root         20G  2.0G   18G  10% /
/dev/mapper/glusterfs-data02  5.0G   33M  5.0G   1% /data02
/dev/mapper/glusterfs-data01  5.0G   33M  5.0G   1% /data01
127.0.0.1:/gv1                 20G  334M   20G   2% /mnt    # 容量为3/4，相当于raid5
```

### 磁盘存储的平衡
平衡布局是很有必要的，因为布局结构是静态的，当新的 bricks 加入现有卷，新创建的文件会分布到旧的 bricks 中，所以需要平衡布局结构，使新加入的 bricks 生效。布局平衡只是使新布局生效，并不会在新的布局中移动老的数据，如果你想在新布局生效后，重新平衡卷中的数据，还需要对卷中的数据进行平衡。
```
# 创建一个分散卷
[root@glusterfs1 ~]# gluster volume create gv1 disperse 3 glusterfs1:/data01 glusterfs2:/data01 glusterfs3:/data01 force   
volume create: gv1: success: please start the volume to access data
[root@glusterfs1 ~]# gluster volume start gv1
volume start: gv1: success
[root@glusterfs1 ~]# mount -t glusterfs 127.0.0.1:/gv1 /mnt
# 创建三个10M的文件
[root@glusterfs1 mnt]# dd if=/dev/zero bs=1024 count=10000 of=/mnt/10M-1.file
10000+0 records in
10000+0 records out
10240000 bytes (10 MB) copied, 2.3449 s, 4.4 MB/s
[root@glusterfs1 mnt]# dd if=/dev/zero bs=1024 count=10000 of=/mnt/10M-2.file
10000+0 records in
10000+0 records out
10240000 bytes (10 MB) copied, 2.66975 s, 3.8 MB/s
[root@glusterfs1 mnt]# dd if=/dev/zero bs=1024 count=10000 of=/mnt/10M-3.file
10000+0 records in
10000+0 records out
10240000 bytes (10 MB) copied, 2.11298 s, 4.8 MB/s

[root@glusterfs1 mnt]# ll -h /data01/
total 15M
-rw-r--r-- 2 root root 4.9M Aug 19 17:27 10M-1.file
-rw-r--r-- 2 root root 4.9M Aug 19 17:27 10M-2.file
-rw-r--r-- 2 root root 4.9M Aug 19 17:27 10M-3.file

[root@glusterfs2 ~]# ll -h /data01/
total 15M
-rw-r--r-- 2 root root 4.9M Aug 19 17:27 10M-1.file
-rw-r--r-- 2 root root 4.9M Aug 19 17:27 10M-2.file
-rw-r--r-- 2 root root 4.9M Aug 19 17:27 10M-3.file

[root@glusterfs3 ~]# ll -h /data01/
total 15M
-rw-r--r-- 2 root root 4.9M Aug 19 17:27 10M-1.file
-rw-r--r-- 2 root root 4.9M Aug 19 17:27 10M-2.file
-rw-r--r-- 2 root root 4.9M Aug 19 17:27 10M-3.file

# 新加三个盘进来做分布式分散卷
[root@glusterfs1 /]# umount /mnt/
[root@glusterfs1 ~]# gluster volume stop gv1
Stopping volume will make its data inaccessible. Do you want to continue? (y/n) y
volume stop: gv1: success
[root@glusterfs1 ~]# gluster volume add-brick gv1 glusterfs1:/data02 glusterfs2:/data02 glusterfs3:/data02 force
volume add-brick: success
[root@glusterfs1 ~]# gluster volume start gv1 
volume start: gv1: success
[root@glusterfs1 ~]# mount -t glusterfs 127.0.0.1:/gv1 /mnt

# 再创建三个文件
[root@glusterfs1 ~]# dd if=/dev/zero bs=1024 count=10000 of=/mnt/10M-4.file
10000+0 records in
10000+0 records out
10240000 bytes (10 MB) copied, 1.90685 s, 5.4 MB/s
[root@glusterfs1 ~]# dd if=/dev/zero bs=1024 count=10000 of=/mnt/10M-5.file
10000+0 records in
10000+0 records out
10240000 bytes (10 MB) copied, 1.99968 s, 5.1 MB/s
[root@glusterfs1 ~]# dd if=/dev/zero bs=1024 count=10000 of=/mnt/10M-6.file
10000+0 records in
10000+0 records out
10240000 bytes (10 MB) copied, 2.3427 s, 4.4 MB/s

# 文件分布不均匀
[root@glusterfs1 ~]# ll /data01/
total 25000
-rw-r--r-- 2 root root 5120000 Aug 19 17:27 10M-1.file
-rw-r--r-- 2 root root 5120000 Aug 19 17:27 10M-2.file
-rw-r--r-- 2 root root 5120000 Aug 19 17:27 10M-3.file
-rw-r--r-- 2 root root 5120000 Aug 19 17:39 10M-4.file
-rw-r--r-- 2 root root 5120000 Aug 19 17:40 10M-5.file
[root@glusterfs1 ~]# ll /data02/
total 5000
-rw-r--r-- 2 root root 5120000 Aug 19 17:40 10M-6.file

# 进行磁盘存储平衡
[root@glusterfs1 ~]# gluster volume rebalance gv1 start
volume rebalance: gv1: success: Rebalance on gv1 has been started successfully. Use rebalance status command to check status of the rebalance process.
ID: 96dee621-7bab-49a2-9172-1ba5cef90311
[root@glusterfs1 ~]# gluster volume rebalance gv1 status
                                    Node Rebalanced-files          size       scanned      failures       skipped               status  run time in h:m:s
                               ---------      -----------   -----------   -----------   -----------   -----------         ------------     --------------
                              glusterfs2                0        0Bytes             6             0             0            completed        0:00:05
                              glusterfs3                0        0Bytes             5             0             0            completed        0:00:05
                               localhost                1         9.8MB             6             0             0            completed        0:00:05
volume rebalance: gv1: success

# 再次查看文件分布
[root@glusterfs1 ~]# ll -h /data01/
total 20M
-rw-r--r-- 2 root root 4.9M Aug 19 17:27 10M-1.file
-rw-r--r-- 2 root root 4.9M Aug 19 17:27 10M-3.file
-rw-r--r-- 2 root root 4.9M Aug 19 17:39 10M-4.file
-rw-r--r-- 2 root root 4.9M Aug 19 17:40 10M-5.file
[root@glusterfs1 ~]# ll -h /data02/
total 9.8M
-rw-r--r-- 2 root root 4.9M Aug 19 17:27 10M-2.file
-rw-r--r-- 2 root root 4.9M Aug 19 17:40 10M-6.file

#从上面可以看出部分文件已经平衡到新加入的brick中了
```
> 每做一次扩容后都需要做一次磁盘平衡。 磁盘平衡是在万不得已的情况下再做的，一般再创建一个卷就可以了。

### 移除brick
注意：当你移除 bricks 的时候，你在 gluster 的挂载点将不能继续访问数据，只有配置文件中的信息移除后你才能继续访问 bricks 中的数据。当移除分布式复制卷或者分布式条带卷的时候，移除的 bricks 数目必须是 replica 或者 disperse 的倍数。
```
# 卷原有信息
[root@glusterfs1 ~]# gluster volume info 
Volume Name: gv1
Type: Distributed-Disperse
Volume ID: d01bb6ad-fcc9-4051-a17a-6bef21f52440
Status: Started
Snapshot Count: 0
Number of Bricks: 2 x (2 + 1) = 6
Transport-type: tcp
Bricks:
Brick1: glusterfs1:/data01
Brick2: glusterfs2:/data01
Brick3: glusterfs3:/data01
Brick4: glusterfs1:/data02
Brick5: glusterfs2:/data02
Brick6: glusterfs3:/data02
Options Reconfigured:
transport.address-family: inet
nfs.disable: on

[root@glusterfs1 ~]# gluster volume stop gv1
Stopping volume will make its data inaccessible. Do you want to continue? (y/n) y
volume stop: gv1: success

# 移除brick
[root@glusterfs1 ~]# gluster volume remove-brick gv1 glusterfs1:/data01 glusterfs2:/data01 glusterfs3:/data01 force
Remove-brick force will not migrate files from the removed bricks, so they will no longer be available on the volume.
Do you want to continue? (y/n) y
volume remove-brick commit force: success

# 移除后卷信息
[root@glusterfs1 ~]# gluster volume info 
Volume Name: gv1
Type: Disperse
Volume ID: d01bb6ad-fcc9-4051-a17a-6bef21f52440
Status: Stopped
Snapshot Count: 0
Number of Bricks: 1 x (2 + 1) = 3
Transport-type: tcp
Bricks:
Brick1: glusterfs1:/data02
Brick2: glusterfs2:/data02
Brick3: glusterfs3:/data02
Options Reconfigured:
performance.client-io-threads: on
transport.address-family: inet
nfs.disable: on

[root@glusterfs1 ~]# gluster volume start gv1 
volume start: gv1: success
[root@glusterfs1 ~]# mount -t glusterfs 127.0.0.1:/gv1 /mnt
[root@glusterfs1 ~]# ll /mnt/
total 20000
-rw-r--r-- 1 root root 10240000 Aug 19 17:27 10M-2.file
-rw-r--r-- 1 root root 10240000 Aug 19 17:40 10M-6.file
```
### 删除卷
```
[root@glusterfs1 ~]# umount /mnt/
[root@glusterfs1 ~]# gluster volume stop gv1
Stopping volume will make its data inaccessible. Do you want to continue? (y/n) y
volume stop: gv1: success
[root@glusterfs1 ~]# gluster volume delete gv1
Deleting volume will erase all information about the volume. Do you want to continue? (y/n) y
volume delete: gv1: success

# 文件还在
[root@glusterfs1 ~]# ll /data01/
total 20000
-rw-r--r-- 2 root root 5120000 Aug 19 17:27 10M-1.file
-rw-r--r-- 2 root root 5120000 Aug 19 17:27 10M-3.file
-rw-r--r-- 2 root root 5120000 Aug 19 17:39 10M-4.file
-rw-r--r-- 2 root root 5120000 Aug 19 17:40 10M-5.file
[root@glusterfs1 ~]# ll /data02/
total 10000
-rw-r--r-- 2 root root 5120000 Aug 19 17:27 10M-2.file
-rw-r--r-- 2 root root 5120000 Aug 19 17:40 10M-6.file

```
## glusterfs文件系统优化
参数项目 | 说明 | 缺省值 | 合法值
-|-|-|-|-
Auth_allow | IP访问授权 | *.allow all | Ip地址
Cluster.min-free-disk | 剩余磁盘空间阀值 | 10% | 百分比
Cluster.stripe-block-size | 条带大小 | 128KB | 字节
Network.frame-timeout | 请求等待时间 | 1800s | 1-1800
Network.ping-timeout | 客户端等待时间 | 42s | 0-42
Nfs.disabled | 关闭NFS服务 | Off | Off\on
Performance.io-thread-count | IO线程数 | 16 | 0-65
Performance.cache-refresh-timeout | 缓存校验时间 | 1s | 0-61
Performance.cache-size | 读缓存大小 | 32MB | 字节

- Performance.quick-read: #优化读取小文件的性能
- Performance.read-ahead: #用预读的方式提高读取的性能，有利于应用频繁持续性的访问文件，当应用完成当前数据块读取的时候，下一个数据块就已经准备好了。
- Performance.write-behind: #先写入缓存内，在写入硬盘，以提高写入的性能。
- Performance.io-cache: #缓存已经被读过的。
### 优化参数调整方式
- 命令格式：
    - gluster.volume set <卷><参数>

- 打开预读方式访问存储
```
[root@glusterfs1 ~]# gluster volume set gv1 performance.read-ahead on
volume set: success
```
- 调整读取缓存的大小
```
[root@glusterfs1 ~]# gluster volume set gv1 performance.cache-size 256MB
volume set: success
```
- 查看具体有哪些参数可以优化
```
[root@glusterfs1 ~]# gluster volume set help
```
### 监控及日常维护
使用zabbix自带的模板即可，CPU、内存、磁盘空间、主机运行时间、系统load。日常情况要查看服务器监控值，遇到报警要及时处理。
```
# 看下节点有没有在线
gluster volume status nfsp

# 启动完全修复
gluster volume heal gv1 full

# 查看需要修复的文件
gluster volume heal gv1 info

# 查看修复成功的文件
gluster volume heal gv1 info healed

# 查看修复失败的文件
gluster volume heal gv1 heal-failed

# 查看主机的状态
gluster peer status

# 查看脑裂的文件
gluster volume heal gv1 info split-brain

# 激活quota功能
gluster volume quota gv1 enable

# 关闭quota功能
gulster volume quota gv1 disable

# 目录限制（卷中文件夹的大小）
gluster volume quota limit-usage /data/30MB --/gv1/data

# quota信息列表
gluster volume quota gv1 list

# 限制目录的quota信息
gluster volume quota gv1 list /data

# 设置信息的超时时间
gluster volume set gv1 features.quota-timeout 5

# 删除某个目录的quota设置
gluster volume quota gv1 remove /data

备注：quota功能，主要是对挂载点下的某个目录进行空间限额。如：/mnt/gulster/data目录，而不是对组成卷组的空间进行限制。
```

## 模拟故障
### 模拟硬盘故障
```
# 新建一个gv1
[root@glusterfs1 ~]# gluster volume create gv1 replica 2 glusterfs1:/data01 glusterfs2:/data02 force
volume create: gv1: success: please start the volume to access data
[root@glusterfs1 ~]# gluster volume start gv1
volume start: gv1: success
[root@glusterfs1 ~]# mount -t glusterfs 127.0.0.1:/gv1 /mnt
# 写入数据
[root@glusterfs1 ~]# ll -h /mnt/
total 20M
-rw-r--r-- 1 root root 9.8M Aug 20 15:34 10M-1.file
-rw-r--r-- 1 root root 9.8M Aug 20 15:34 10M-2.file

#删除一块盘再加入一块盘
[root@glusterfs1 ~]# gluster volume status
Status of volume: gv1
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick glusterfs1:/data01                    49152     0          Y       1603 
Brick glusterfs2:/data02                    49152     0          Y       1455 
Self-heal Daemon on localhost               N/A       N/A        Y       1624 
Self-heal Daemon on glusterfs2              N/A       N/A        Y       1476 
Self-heal Daemon on glusterfs3              N/A       N/A        Y       1456 
Task Status of Volume gv1
------------------------------------------------------------------------------
There are no active volume tasks

# 杀掉原有的进程
[root@glusterfs1 ~]# kill -15 1603
[root@glusterfs1 ~]# gluster volume status
Status of volume: gv1
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick glusterfs1:/data01                    N/A       N/A        N       N/A  
Brick glusterfs2:/data02                    49152     0          Y       1455 
Self-heal Daemon on localhost               N/A       N/A        Y       1624 
Self-heal Daemon on glusterfs3              N/A       N/A        Y       1456 
Self-heal Daemon on glusterfs2              N/A       N/A        Y       1476 
Task Status of Volume gv1
------------------------------------------------------------------------------
There are no active volume tasks

[root@glusterfs1 ~]# mkfs.xfs /dev/sdb
meta-data=/dev/sdb               isize=512    agcount=4, agsize=327680 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=1310720, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@glusterfs1 ~]# mount /dev/sdb /data01/
[root@glusterfs1 ~]# ll /data01/
total 0

# 在挂载点上创建一个尚不存在的目录，然后删除该目录，通过执行setfattr对元数据更改日志执行相同操作，触发自愈
[root@glusterfs2 ~]# mkdir /mnt/1
[root@glusterfs2 ~]# rmdir /mnt/1
[root@glusterfs2 ~]# setfattr -n trusted.non-existent-key -v abc /mnt
[root@glusterfs2 ~]# setfattr -x trusted.non-existent-key  /mnt
# 查看目录属性
[root@glusterfs2 ~]# getfattr -d -m. -e hex /data02/
getfattr: Removing leading '/' from absolute path names
# file: data02/
trusted.afr.dirty=0x000000000000000000000000
trusted.afr.gv1-client-0=0x000000000000000200000002
trusted.gfid=0x00000000000000000000000000000001
trusted.glusterfs.mdata=0x010000000000000000000000005d5bacd80000000014243216000000005d5baccb000000000bfc4c3a00000000000000000000000000000000
trusted.glusterfs.volume-id=0xc888433b65fe443f9d0ef91b9a0f7b72

[root@glusterfs1 ~]# gluster volume heal gv1 info
Brick glusterfs1:/data01
Status: Transport endpoint is not connected
Number of entries: -

Brick glusterfs2:/data02
/ 
Status: Connected
Number of entries: 1
# 将新的节点/data02替换/data01
[root@glusterfs1 ~]# gluster volume replace-brick gv1 glusterfs1:/data01 glusterfs1:/data02 commit force
volume replace-brick: success: replace-brick commit force operation successful

[root@glusterfs1 ~]# gluster volume status
Status of volume: gv1
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick glusterfs1:/data02                    49152     0          Y       1855 
Brick glusterfs2:/data02                    49152     0          Y       1455 
Self-heal Daemon on localhost               N/A       N/A        Y       1865 
Self-heal Daemon on glusterfs3              N/A       N/A        Y       1517 
Self-heal Daemon on glusterfs2              N/A       N/A        Y       1765 
Task Status of Volume gv1
------------------------------------------------------------------------------
There are no active volume tasks

[root@glusterfs1 ~]# getfattr -d -m. -e hex /data02/
getfattr: Removing leading '/' from absolute path names
# file: data02/
trusted.afr.dirty=0x000000000000000000000000
trusted.gfid=0x00000000000000000000000000000001
trusted.glusterfs.mdata=0x010000000000000000000000005d5bacd80000000014243216000000005d5baccb000000000bfc4c3a00000000000000000000000000000000
trusted.glusterfs.volume-id=0xc888433b65fe443f9d0ef91b9a0f7b72

# 同步完成，查看文件是否已同步过来
[root@glusterfs1 ~]# ll /data02/
total 20000
-rw-r--r-- 2 root root 10240000 Aug 20 15:34 10M-1.file
-rw-r--r-- 2 root root 10240000 Aug 20 15:34 10M-2.file
[root@glusterfs1 ~]# gluster volume heal gv1 info
Brick glusterfs1:/data02
Status: Connected
Number of entries: 0

Brick glusterfs2:/data02
Status: Connected
Number of entries: 0
```
### 模拟主机故障
- 物理故障
    - 同时有多块硬盘故障，数据丢失
    - 系统损坏不可修复
找一台完全一样的机器，至少要保证硬盘数量和大小一致，安装系统，配置和故障机同样的IP，安装 gluster 软件，保证配置一样，在其他健康节点上执行命令 gluster peer status，查看故障服务器的 uuid
```
# 在其他机器上查看glusterfs1的uuid
[root@glusterfs2 ~]# gluster peer status
Number of Peers: 2

Hostname: glusterfs1
Uuid: c2ee8d16-04c7-48d9-a0cb-f6bdd3f96b2c
State: Peer in Cluster (Connected)

Hostname: glusterfs3
Uuid: a500fa96-c84a-42f7-802f-3b9b29887f3d
State: Peer in Cluster (Connected)
# 更改uuid
[root@glusterfs1 ~]# cat /var/lib/glusterd/glusterd.info 
UUID=c2ee8d16-04c7-48d9-a0cb-f6bdd3f96b2c   # 修改成原来的uuid
operating-version=60000

# 在信任存储池中任意节点执行，就会自动开始同步，但在同步的时候会影响整个系统的性能
[root@glusterfs2 ~]# gluster volume heal gv1 full
Launching heal operation to perform full self heal on volume gv1 has been successful 
Use heal info commands to check status.

# 可以查看状态
[root@glusterfs2 ~]# gluster volume heal gv1 info
Brick glusterfs1:/data02
Status: Connected
Number of entries: 0

Brick glusterfs2:/data02
Status: Connected
Number of entries: 0
```
## glusterFS在企业中应用场景
理论和实践分析，GlusterFS目前主要使用大文件存储场景，对于小文件尤其是海量小文件，存储效率和访问性能都表现不佳，海量小文件LOSF问题是工业界和学术界的人工难题，GlusterFS作为通用的分布式文件系统，并没有对小文件额外的优化措施，性能不好也是可以理解的。
- Media
    - 文档、图片、音频、视频
- *Shared storage　　
    - 云存储、虚拟化存储、HPC（高性能计算）
- *Big data
    - 日志文件、RFID（射频识别）数据
