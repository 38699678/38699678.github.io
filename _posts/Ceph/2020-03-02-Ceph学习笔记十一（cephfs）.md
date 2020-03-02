## Ceph学习笔记十一（cephfs使用）  

Ceph文件系统或CephFS是在Ceph的分布式对象存储RADOS之上构建的POSIX兼容文件系统。   
CephFS致力于为各种应用程序提供最新，多用途，高可用性和高性能的文件存储，包括传统存储（如共享主目录，HPC暂存空间和分布式工作流共享存储）。 CephFS通过使用一些新颖的架构选择来实现这些目标。值得注意的是，文件元数据与文件数据存储在单独的存储池中，并通过可调整大小的元数据服务器或MDS集群提供服务，该集群可扩展以支持更高吞吐量的元数据工作负载。文件系统的客户端可以直接访问RADOS以读取和写入文件数据块。因此，工作负载可能会随着基础RADOS对象存储的大小线性扩展。也就是说，没有网关或代理为客户端中介数据I / O。 通过MDS集群协调对数据的访问，该集群充当客户端和MDS协作维护的分布式元数据缓存状态的授权机构。每个MDS都会将对元数据的汇总为对RADOS上日记的一系列有效写入。 MDS不会在本地存储任何元数据状态。此模型允许在POSIX文件系统的上下文中客户端之间进行连贯且快速的协作。

MDS介绍：  
- MDS全称Ceph Metadata Server，是CephFS服务依赖的元数据服务。
- 元数据的内存缓存，为了加快元数据的访问。
- 保存了文件系统的元数据(对象里保存了子目录和子文件的名称和inode编号)
- 保存cephfs日志journal，日志是用来恢复mds里的元数据缓存
- 重启mds的时候会通过replay的方式从osd上加载之前缓存的元数据
- 对外提供服务只有一个active mds。
- 所有用户的请求都只落在一个active mds上。  

![avatar](https://docs.ceph.com/docs/master/_images/cephfs-architecture.svg) 

### 创建一个ceph file system  
1. 创建两个存储池，需要建立两个存储池。一个存储池用来保存元数据，一个存储池用来保存数据文件。
``` bash  
#ceph-deploy  mds create {node.name}
$ ceph osd pool create cephfs_data
$ ceph osd pool create cephfs_metadata
```
2. 创建一个ceph文件系统。
``` bash
$ ceph fs new <fs_name> <metadata> <data>
$ ceph fs new cephfs cephfs_metadata cephfs_data
```
3. 查看ceph文件系统
``` bash
[root@lspre-ceph-1 ~]# ceph fs ls
name: db_backup, metadata pool: db_backup_metadata, data pools: [db_backup_pool ]
[root@lspre-ceph-1 ~]# ceph mds stat
db_backup-1/1/1 up  {0=lspre-ceph-3=up:active}
```

创建完文件系统，mds也被激活了。你可以尝试去mount文件系统。

4. 挂载cephfs  
-  安装ceph-common  
   在挂载节点上安装ceph-common  
   #yum install -y ceph-common  
-  在挂载节点创建token文件  
   #vi /etc/ceph/admin.key   
   AQC/mo9VxqsXDBAAQ/LQtTmR+GTPs65KBsEPrw==  
-  通过mount挂载  
   ``` bash
   #mount -t ceph {ip-address-of-monitor}:6789:/ /mnt/mycephfs/ -o name=admin，secretfile=/etc/ceph/admin.key
   ```
   ip-address-of-monitor: 此为ceph集群的任意monitor节点ip  
   name=admin ：此为你token绑定的用户  
   secretfile=/etc/ceph/admin.key ： 指定token文件
-  编辑/etc/fstab 开机自动挂载
   ``` bash
   #vi /etc/fstab
   172.18.178.90:6789:/     /db-backup    ceph    name=admin,secretfile=/etc/ceph/admin.key,noatime,_netdev 0 2
   ```

### 管理命令
- 创建文件系统  
``` bash
#ceph fs new <file system name> <metadata pool name> <data pool name>
```
- 查看文件系统
``` bash
[root@ceph-master ceph-cluster]# ceph fs ls
name: db_backup, metadata pool: db_backup_metadata, data pools: [db_backup_pool ]
```
- 查看信息信息
``` bash
[root@ceph-master ceph-cluster]# ceph fs dump
dumped fsmap epoch 4
e4
enable_multiple, ever_enabled_multiple: 0,0
compat: compat={},rocompat={},incompat={1=base v0.20,2=client writeable ranges,3=default file layouts on dirs,4=dir inode in separate object,5=mds uses versioned encoding,6=dirfrag is stored in omap,8=no anchor table,9=file layout v2,10=snaprealm v2}
legacy client fscid: 1
 
Filesystem 'db_backup' (1)
fs_name	db_backup
epoch	4
flags	12
created	2020-02-28 21:05:38.511029
modified	2020-02-28 21:05:39.518327
tableserver	0
root	0
session_timeout	60
session_autoclose	300
max_file_size	1099511627776
min_compat_client	-1 (unspecified)
last_failure	0
last_failure_osd_epoch	0
compat	compat={},rocompat={},incompat={1=base v0.20,2=client writeable ranges,3=default file layouts on dirs,4=dir inode in separate object,5=mds uses versioned encoding,6=dirfrag is stored in omap,8=no anchor table,9=file layout v2,10=snaprealm v2}
max_mds	1
in	0
up	{0=494178}
failed	
damaged	
stopped	
data_pools	[12]
metadata_pool	13
inline_data	disabled
balancer	
standby_count_wanted	0
494178:	10.5.20.203:6808/1465108103 'lspre-ceph-3' mds.0.3 up:active seq 44

```
- 删除文件系统
``` bash
#ceph fs rm <file system name> [--yes-i-really-really-mean-it]
```

- 修改文件系统参数值
``` bash
#ceph fs set <file system name> <var> <val>
#ceph fs set --help
fs set <fs_name> max_mds|max_file_size|allow_new_snaps|inline_data|cluster_down|allow_dirfrags|balancer|standby_  set fs parameter <var> to <val> count_wanted|session_timeout|session_autoclose|down|joinable|min_compat_client <val> {<confirm>}                 
fs set-default <fs_name>   set the default to the named filesystem

设置文件系统大小
# ceph fs set <fs name> max_file_size <size in bytes>
#

```

- 添加pool
```
# ceph fs add_data_pool <file system name> <pool name/id>
```

### 添加/删除mds
在master节点中/etc/ceph-cluster 
``` bash 
#ceph-deploy mds create lspre-ceph-2    
#ceph mds stat  
db_backup-1/1/1 up  {0=lspre-ceph-3=up:active}, 1 up:standby  
```

- 删除mds节点
1. 登陆mds节点
``` bash
#systemctl stop ceph-mds@lspre-ceph-2.service
# ceph mds stat
db_backup-1/1/1 up  {0=lspre-ceph-3=up:active}
```
2. 删除mds的认证信息
``` bash
# ceph auth del mds.lspre-ceph-2
updated
```
3. 删除其他相关信息
``` bash
# systemctl disable ceph-mds@lspre-ceph-2
Removed symlink /etc/systemd/system/ceph-mds.target.wants/ceph-mds@lspre-ceph-2.service.
# rm -rf /var/lib/ceph/mds/ceph-lspre-ceph-2
```
4. 查看ceph状态
``` bash
   [root@lspre-ceph-2 mds]# ceph -s 
  cluster:
    id:     2e48a61c-02d9-44f3-bb55-ceb92a81fb86
    health: HEALTH_WARN
            insufficient standby MDS daemons available
 
  services:
    mon: 5 daemons, quorum idc-pub-ceph-01,idc-pub-ceph-02,idc-pub-ceph-03,lspre-ceph-1,lspre-ceph-2
    mgr: ceph-master(active), standbys: lspre-ceph-2, lspre-ceph-1
    mds: db_backup-1/1/1 up  {0=lspre-ceph-3=up:active}
    osd: 12 osds: 12 up, 12 in
 
  data:
    pools:   6 pools, 192 pgs
    objects: 727.1 k objects, 2.7 TiB
    usage:   8.2 TiB used, 15 TiB / 23 TiB avail
    pgs:     192 active+clean
 
  io:
    client:   5.2 MiB/s wr, 0 op/s rd, 25 op/s wr
```
