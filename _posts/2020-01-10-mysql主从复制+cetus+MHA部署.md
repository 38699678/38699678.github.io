# cetus+MHA+mysql主从部署
### mysql安装  
- 本文mysql数据使用的是mysql8版本。  
``` bash  
	#wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-8.0.17-el7-x86_64.tar.gz  
	#tar -zxvf mysql-8.0.17-el7-x86_64.tar.gz  
	#mv mysql-8.0.17-el7-x86_64 mysql  
	#cd myql  
	#mkdir -p /data/  
	#groupadd mysql && useradd -r -g mysql -s /bin/false mysql  
	#vi /usr/local/mysql/mysql${port}.cnf
	[client]
	port    = 6606
	socket  = /data/mysql_6606/mysql.sock

	[mysql]
	prompt="\u@mysqldb \R:\m:\s [\d]> "
	no-auto-rehash

	[mysqld]
	user    = mysql
	port    = 6606
	mysqlx_port = 6607
	basedir = /usr/local/mysql
	datadir = /data/mysql_6606
	socket  = /data/mysql_6606/mysql.sock
	pid-file = /data/mysql_6606/mysqldb.pid
	character-set-server = utf8mb4
	#字符区分大小写
	collation-server = utf8mb4_0900_as_cs
	skip_name_resolve = 1
	#
	default-authentication-plugin=mysql_native_password
	#表名不区分大小写
	lower-case-table-names = 1
	autocommit = 1
	group_concat_max_len = 10240

	range_optimizer_max_mem_size=0
	innodb_adaptive_hash_index=0
	table_open_cache=25000
	innodb_status_output=0

	#############主从复制
	sql_mode='STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION'
	gtid-mode=on
	enforce-gtid-consistency=on
	log-slave-updates=on
	slave-parallel-type=LOGICAL_CLOCK
	slave-parallel-workers=8
	log_bin_trust_function_creators=1
	###################


	open_files_limit    = 65535
	back_log = 1024
	max_connections = 8000
	max_connect_errors = 1000000
	#table_open_cache = 1024
	table_definition_cache = 1024
	table_open_cache_instances = 64
	thread_stack = 512K
	external-locking = FALSE
	max_allowed_packet = 128M
	sort_buffer_size = 4M
	join_buffer_size = 4M
	thread_cache_size = 768
	interactive_timeout = 28800
	wait_timeout = 28800
	tmp_table_size = 32M
	max_heap_table_size = 32M
	slow_query_log = 1
	log_timestamps = SYSTEM
	slow_query_log_file = /data/mysql_6606/slow.log
	log-error = /data/mysql_6606/error.log
	long_query_time = 2
	log_queries_not_using_indexes =1
	log_throttle_queries_not_using_indexes = 60
	min_examined_row_limit = 100
	log_slow_admin_statements = 1
	log_slow_slave_statements = 1
	server-id = 6606
	log-bin = /data/mysql_6606/mybinlog
	sync_binlog = 1
	binlog_cache_size = 4M
	max_binlog_cache_size = 2G
	max_binlog_size = 1G
	master_info_repository = TABLE
	relay_log_info_repository = TABLE
	gtid_mode = on
	enforce_gtid_consistency = 1
	log_slave_updates
	slave-rows-search-algorithms = 'INDEX_SCAN,HASH_SCAN'
	binlog_format = row
	binlog_checksum = 1
	relay_log_recovery = 1
	relay-log-purge = 1
	key_buffer_size = 32M
	read_buffer_size = 8M
	read_rnd_buffer_size = 4M
	bulk_insert_buffer_size = 64M
	myisam_sort_buffer_size = 128M
	myisam_max_sort_file_size = 10G
	myisam_repair_threads = 1
	lock_wait_timeout = 3600
	explicit_defaults_for_timestamp = 1
	innodb_thread_concurrency = 0
	innodb_sync_spin_loops = 100
	innodb_spin_wait_delay = 30

	transaction_isolation = READ-COMMITTED
	innodb_buffer_pool_size = 100G
	innodb_buffer_pool_instances = 4
	innodb_buffer_pool_load_at_startup = 1
	innodb_buffer_pool_dump_at_shutdown = 1
	innodb_data_file_path = ibdata1:1G:autoextend
	innodb_flush_log_at_trx_commit = 1
	innodb_log_buffer_size = 32M
	innodb_log_file_size = 2G
	innodb_log_files_in_group = 2
	innodb_max_undo_log_size = 4G
	innodb_undo_directory = /data/mysql_6606/undolog

	# 根据您的服务器IOPS能力适当调整
	# 一般配普通SSD盘的话，可以调整到 10000 - 20000
	# 配置高端PCIe SSD卡的话，则可以调整的更高，比如 50000 - 80000
	innodb_io_capacity = 10000
	innodb_io_capacity_max = 15000
	innodb_flush_sync = 0
	innodb_flush_neighbors = 0
	innodb_write_io_threads = 8
	innodb_read_io_threads = 8
	innodb_purge_threads = 4
	innodb_page_cleaners = 4
	innodb_open_files = 65535
	innodb_max_dirty_pages_pct = 50
	innodb_flush_method = O_DIRECT
	innodb_lru_scan_depth = 4000
	innodb_checksum_algorithm = crc32
	innodb_lock_wait_timeout = 10
	innodb_rollback_on_timeout = 1
	innodb_print_all_deadlocks = 1
	innodb_file_per_table = 1
	innodb_online_alter_log_max_size = 4G
	innodb_stats_on_metadata = 0


	# some var for MySQL 8
	log_error_verbosity = 3
	innodb_print_ddl_logs = 1
	binlog_expire_logs_seconds = 259200
	#innodb_dedicated_server = 0

	innodb_status_file = 1
	#注意: 开启 innodb_status_output & innodb_status_output_locks 后, 可能会导致log-error文件增长较快
	innodb_status_output = 0
	innodb_status_output_locks = 0

	#performance_schema
	performance_schema = 1
	performance_schema_instrument = '%memory%=on'
	performance_schema_instrument = '%lock%=on'

	#innodb monitor
	innodb_monitor_enable="module_innodb"
	innodb_monitor_enable="module_server"
	innodb_monitor_enable="module_dml"
	innodb_monitor_enable="module_ddl"
	innodb_monitor_enable="module_trx"
	innodb_monitor_enable="module_os"
	innodb_monitor_enable="module_purge"
	innodb_monitor_enable="module_log"
	innodb_monitor_enable="module_lock"
	innodb_monitor_enable="module_buffer"
	innodb_monitor_enable="module_index"
	innodb_monitor_enable="module_ibuf_system"
	innodb_monitor_enable="module_buffer_page"
	#innodb_monitor_enable="module_adaptive_hash"

	[mysqldump]
	quick
	max_allowed_packet = 32M
	```
- 初始化数据库  
  #cd /usr/local/mysql/bin  
  #./mysqld --defaults-file=/usr/local/mysql/mysql6626.cnf --initialize  
  #查看/data/mysql_6626目录是否有数据文件  
- 修改root密码并启动服务  
  #查看初始化密码  
  #grep password /data/mysql_6626/error.log   

  #mysqld_safe --defaults-file=/usr/local/mysql/mysql6626.cnf &  
  #mysql_secure_installation
- 到此数据库就安装完毕了，其他节点也可以根据这个步骤安装，配置文件要根据实际情况做修改。
### 配置mysql主从模式
- 架构：   
  master：192.168.120.13  6606  
  slave1: 192.168.120.14  6616  
  slave2: 192.168.120.15  6626  
- 注意事项：   
  主从服务器操作系统版本和位数一致；   
  Master 和 Slave 数据库的版本要一致；   
  Master 和 Slave 数据库中的数据要一致；   
  Master 开启二进制日志， Master 和 Slave 的 server_id 在局域网内必须唯一   
- 修改配置文件
  在每个节点部署mysql后，要根据架构去修改数据库配置文件 
  server_id: 必须唯一
  bin_log: 二进制日志必须打开
- master：  
  #create user rpel@'192.168.120.%' idnetified with mysql_native_password by 'mysql123';  
  #grant replication slave on *.* to repl@'192.168.120.%';   
  #flush privileges  
  #show master status;  
  #mysql> show master status;  
    +-----------------+----------+--------------+------------------+-------------------------------------------+
    | File            | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set                         |
    +-----------------+----------+--------------+------------------+-------------------------------------------+
    | mybinlog.000003 |      195 |              |                  | 34e91deb-32d7-11ea-ae86-000c29350c8b:1-14 |
    +-----------------+----------+--------------+------------------+-------------------------------------------+
- slave:  
  #stop slave;  
  #reset slave all;  
  #change master to master_host='192.168.120.13',master_user='repl',master_port=6606,master_password="mysql123",master_auto_position = 1;
  #start slave;  
  ``` bash
    mysql> show slave status\G
    *************************** 1. row ***************************
                Slave_IO_State: Waiting for master to send event
                    Master_Host: 192.168.120.13
                    Master_User: repl
                    Master_Port: 6606
                    Connect_Retry: 60
                Master_Log_File: mybinlog.000003
            Read_Master_Log_Pos: 195
                Relay_Log_File: node3-relay-bin.000006
                    Relay_Log_Pos: 367
            Relay_Master_Log_File: mybinlog.000003
                Slave_IO_Running: Yes
                Slave_SQL_Running: Yes
                Replicate_Do_DB: 
            Replicate_Ignore_DB: 
            Replicate_Do_Table: 
        Replicate_Ignore_Table: 
        Replicate_Wild_Do_Table: 
    Replicate_Wild_Ignore_Table: 
                    Last_Errno: 0
                    Last_Error: 
                    Skip_Counter: 0
            Exec_Master_Log_Pos: 195
                Relay_Log_Space: 575
                Until_Condition: None
                Until_Log_File: 
                    Until_Log_Pos: 0
            Master_SSL_Allowed: No
            Master_SSL_CA_File: 
            Master_SSL_CA_Path: 
                Master_SSL_Cert: 
                Master_SSL_Cipher: 
                Master_SSL_Key: 
            Seconds_Behind_Master: 0
    Master_SSL_Verify_Server_Cert: No
                    Last_IO_Errno: 0
                    Last_IO_Error: 
                Last_SQL_Errno: 0
                Last_SQL_Error: 
    Replicate_Ignore_Server_Ids: 
                Master_Server_Id: 6606
                    Master_UUID: 34e91deb-32d7-11ea-ae86-000c29350c8b
                Master_Info_File: mysql.slave_master_info
                        SQL_Delay: 0
            SQL_Remaining_Delay: NULL
        Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
  ```
### 安装MHA+Cetus
- MHA知识
  HA（Master High Availability）是一套相对成熟的MySQL高可用方案，能做到在0~30s内自动完成数据库的故障切换操作，在master服务器不宕机的情况下，本能保证数据的一致性。  
  1.它由两部分组成：MHA Manager（管理节点）和MHA Node（数据节点）。其中，MHA Manager可以单独部署在一台独立的机器上管理多个master-slave集群，也可以部署在一台slave上。MHA Node则运行在每个mysql节点上，MHA Manager会定时探测集群中的master节点，当master出现故障时，它自动将最新数据的slave提升为master，然后将其它所有的slave指向新的master。
  2.在MHA自动故障切换过程中，MHA试图保存master的二进制日志，从而最大程度地保证数据不丢失，当这并不总是可行的，譬如，主服务器硬件故障或无法通过ssh访问，MHA就没法保存二进制日志，这样就只进行了故障转移但丢失了最新数据。可结合MySQL 5.5中推出的半同步复制来降低数据丢失的风险。  
  MHA软件由两部分组成：Manager工具包和Node工具包，具体说明如下：    
``` text
    MHA Manager：

    1. masterha_check_ssh：检查MHA的SSH配置状况

    2. masterha_check_repl：检查MySQL的复制状况

    3. masterha_manager：启动MHA

    4. masterha_check_status：检测当前MHA运行状态

    5. masterha_master_monitor：检测master是否宕机

    6. masterha_master_switch：控制故障转移（自动或手动）

    7. masterha_conf_host：添加或删除配置的server信息

    8. masterha_stop：关闭MHA
   
    MHA Node:

    save_binary_logs：保存或复制master的二进制日志

    apply_diff_relay_logs：识别差异的relay log并将差异的event应用到其它slave中

    filter_mysqlbinlog：去除不必要的ROLLBACK事件（MHA已不再使用这个工具）

    purge_relay_logs：消除中继日志（不会堵塞SQL线程）
```
- 安装MHA 
  在MHA服务节点和下面每个master和slave节点中都执行安装yum命令。mha4mysql-node-0.58-0.el7.centos.noarch.rpm要在每个节点中安装，mha4mysql-manager-0.58-0.el7.centos.noarch.rpm要在mha的管理节点安装
  ``` bash 
  #rpm -ivh https://mirrors.aliyun.com/epel/epel-release-latest-7.noarch.rpm
  #yum -y install perl perl-DBI perl-DBD-MySQL perl-IO-Socket-SSL perl-Time-HiRes perl-DBD-MySQL perl-Config-Tiny perl-Log-Dispatch perl-Parallel-ForkManager perl-Config-IniFiles  
  #yum install -y mha4mysql-manager-0.58-0.el7.centos.noarch.rpm mha4mysql-node-0.58-0.el7.centos.noarch.rpm  
  #mkdir -p /data/masterha/{bss,rcs,osmc}  
  ```
  **在集群的master和slave节点中，配置无密码登陆，确保在任何一个节点都可以登陆其他节点**
  #ssh-keygen -t rsa  
  #ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.120.3    
  #ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.120.13   
  #ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.120.14    
  #ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.120.15   
  在mysql master中创建监控用户，赋予管理员权限。  
  #create user 'mhaty'@'192.168.120.%' identified with mysql_native_password 'uqnFBzTpmhg4wF3ay1bvQ5o';  
  #grant all on *.* to 'mhaty'@'192.168.120.%';  
  创建配置文件  
    cat /data/masterha/bss.cnf  
  ``` bash
    [server default]
    user=mhaty #此处为mha的监控mysql的用户，需要在数据库中创建
    password="uqnFBzTpmhg4wF3ay1bvQ5o" #用户的密码
    proxy_conf=/data/masterha/bss/bss_cetus.cnf
    manager_workdir=/data/masterha
    manager_log=/data/masterha/bss/bss.log
    remote_workdir=/tmp
    ssh_user=root
    ssh_port=22
    repl_user=repl #主从复制的账号和密码
    repl_password="aaaa" 
    ping_interval=1
    report_script=""
    #此处为配置脚本执行时的参数，主要是主从的信息。-s 为mysql slave地址。
    secondary_check_script=/usr/bin/masterha_secondary_check -s 172.18.178.228 -s 172.18.178.230 --user=root --master_host=172.18.178.226 --master_port=6606 
    shutdown_script=""
    master_ip_failover_script=""
    #以下是配置主从复制的集群信息
    [server_226]
    hostname=172.18.178.226
    master_binlog_dir=/data/mysql_6606
    port=6606

    [server_228]
    hostname=172.18.178.228
    master_binlog_dir=/data/mysql_6616
    port=6616
    candidate_master=1
    check_repl_delay=0

    [server_230]
    hostname=172.18.178.230
    master_binlog_dir=/data/mysql_6626
    port=6626
  ```
  配置mha连接cetus文件
  cat bss/bss_cetus.cnf
  ``` bash
  middle_ipport=192.168.120.3:9606
  middle_user=admin
  middle_pass=admin
  ```
- 部署cetus  
  ``` bash
  #git clone https://github.com/cetus-tools/cetus.git  
  #cd cetus  
  #mkdir build && cd build  
  #CFLAGS='-g -Wpointer-to-int-cast' cmake ../ -DCMAKE_BUILD_TYPE=Debug -DCMAKE_INSTALL_PREFIX=/usr/local/cetus -DSIMPLE_PARSER=ON
  #make install
  ``` 
  #vi cetus/conf/proxy.conf
  ``` text
  [cetus]
    verbose-shutdown=true
    daemon=true
    basedir=/data/cetus
    conf-dir=/data/cetus/conf
    pid-file=/data/cetus/cetus.pid
    plugin-dir=/data/cetus/lib/cetus/plugins
    plugins=proxy,admin
    log-level=debug
    log-file=/data/cetus/logs/cetus.log
    max-open-files=65536
    default-charset=utf8mb4
    #此处表示连接数据库的用户
    default-username=root
    #此处表示连接数据库的默认库，此处必须填写并且必须存在的库
    default-db=test
    ifname=eth0
    default-pool-size=100
    worker-processes=2
    max-resp-size=36485760
    ssl=false
    slave-delay-down=100.000000
    slave-delay-recover=5.000000
    long-query-time=100
    enable-tcp-stream=true
    enable-fast-stream=true
    sql-log-bufsize=10485760
    sql-log-switch=OFF
    sql-log-prefix=cetus
    sql-log-path=/data/cetus/logs
    sql-log-maxsize=1024
    sql-log-mode=BACKEND
    #cetus监听的连接端口
    proxy-address=0.0.0.0:3606
    #后端slave的数据库
    proxy-read-only-backend-addresses=192.168.120.15:6616,192.168.120.14:6626
    #后端master的数据库
    proxy-backend-addresses=192.168.120.13:6606
    #cetus管理接口
    admin-address=0.0.0.0:9606
    #登陆管理接口的用户名和密码
    admin-username=admin
    admin-password=admin
  ```
  #vi cetus/users.json 
  users.json用来配置用户登陆信息，采用键值对的结构，其中键是固定的，值是用户在MySQL创建的登陆用户名和密码。其中user的值是用户名；client_pwd的值是前端登录Cetus的密码;server_pwd的值是Cetus登录后端的密码。  
  ``` text
  {
	"users":	[{
			"user":	"root",
			"client_pwd":	"cetus_app",
			"server_pwd":	"mysql123"
		}, {
			"user":	"test",
			"client_pwd":	"cetus_app1",
			"server_pwd":	"mysql123"
		}]
  } 
  ``` 
  1.cetus文件替换MHA文件  
  进入cetus安装目录
  使用 mha_ld/src 替换所有文件/usr/share/perl5/vendor_perl/MHA/目录的所有同名文件  
  使用 mha_ld/masterha_secondary_check替换masterha_secondary_check命令   
  #/usr/bin/masterha_secondary_check  
  #rm /usr/bin/masterha_secondary_check  
  #cd /usr/bin/  
  上传修改后的masterha_secondary_check  
  #chmod +x /usr/bin/masterha_secondary_check  
  2.检查mha和mysql集群的连通性  
  manager节点检查ssh连通性  
  #masterha_check_ssh --conf=/data/masterha/bss.cnf  
  manager节点检查主从复制状态  
  #masterha_check_repl --conf=/data/masterha/bss.cnf  
- 启动mha和cetus  
  启动mha  
  #masterha_manager --conf=/data/masterha/bss.cnf  --ignore_last_failover &  
  启动cetus      
  bin/cetus --defaults-file=conf/proxy.conf &  
- 测试cetus  
  #mysql -h127.0.0.1 -uadmin -padmin -P9606  
  ``` mysql
    +-------+-------------+---------------------+---------+------+-----------------+------------+------------+-------------+
    | PID   | backend_ndx | address             | state   | type | slave delay(ms) | idle_conns | used_conns | total_conns |
    +-------+-------------+---------------------+---------+------+-----------------+------------+------------+-------------+
    | 29030 | 1           | 192.168.120.13:6606 | up      | rw   | NULL            | 100        | 0          | 100         |
    | 29030 | 2           | 192.168.120.15:6616 | unknown | ro   | 0               | 100        | 0          | 100         |
    | 29030 | 3           | 192.168.120.14:6626 | unknown | ro   | 0               | 100        | 0          | 100         |
  ```

  #mysql -h192.168.120.3  -u root  -pcetus_app  -P 3606    
  ``` bash  
  MySQL [(none)]> show databases;    
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | mysql              |
    | performance_schema |
    | sys                |
    | test               |
    | test1              |
    +--------------------+
    6 rows in set (0.00 sec)
  ```
- 测试mha
  当前主库和从库都处于正常状态，mha会定时扫描和检测主从的状态，以及复制的状态。
  1. 关闭master（192.168.120.13）的数据库。
  2. 查看hma的日志输出
  #cat bss/bss.log
  ``` bash
  Sat Jan 11 00:19:05 2020 - [warning] At least one of monitoring servers is not reachable from this script. This is likely a network problem. Failover should not happen.
  Sat Jan 11 00:19:06 2020 - [warning] Got error on MySQL connect: 2003 (Can't connect to MySQL server on '192.168.120.13' (111))
  Sat Jan 11 00:19:06 2020 - [warning] Connection failed 2 time(s)..
	Sat Jan 11 00:19:07 2020 - [warning] Got error on MySQL connect: 2003 (Can't connect to MySQL server on '192.168.120.13' (111))
	Sat Jan 11 00:19:07 2020 - [warning] Connection failed 3 time(s)..
	Sat Jan 11 00:19:08 2020 - [warning] Got error on MySQL connect: 2003 (Can't connect to MySQL server on '192.168.120.13' (111))
	Sat Jan 11 00:19:08 2020 - [warning] Connection failed 4 time(s)..
	Sat Jan 11 00:19:08 2020 - [warning] Secondary network check script returned errors. Failover should not start so checking server status again. Check network settings for details.
	Sat Jan 11 00:19:09 2020 - [warning] Got error on MySQL connect: 2003 (Can't connect to MySQL server on '192.168.120.13' (111))
	Sat Jan 11 00:19:09 2020 - [warning] Connection failed 1 time(s)..
	Sat Jan 11 00:19:09 2020 - [info] Executing secondary network check script: /usr/bin/masterha_secondary_check -s 192.168.120.14 -s 192.168.120.15 --user=root --master_host=192.168.120.13 --master_port=6606  --user=root  --mas
	ter_host=192.168.120.13  --master_ip=192.168.120.13  --master_port=6606 --master_user=mhaty --master_password=mysql123 --ping_type=SELECT
	Sat Jan 11 00:19:09 2020 - [info] Executing SSH check script: exit 0
	Sat Jan 11 00:19:09 2020 - [info] HealthCheck: SSH to 192.168.120.13 is reachable.
	Monitoring server 192.168.120.14 is reachable, Master is not reachable from 192.168.120.14. OK.
	Monitoring server 192.168.120.15 is reachable, Master is not reachable from 192.168.120.15. OK.
	Sat Jan 11 00:19:09 2020 - [info] Master is not reachable from all other monitoring servers. Failover should start.
	Sat Jan 11 00:19:10 2020 - [warning] Got error on MySQL connect: 2003 (Can't connect to MySQL server on '192.168.120.13' (111))
	Sat Jan 11 00:19:10 2020 - [warning] Connection failed 2 time(s)..
	Sat Jan 11 00:19:11 2020 - [warning] Got error on MySQL connect: 2003 (Can't connect to MySQL server on '192.168.120.13' (111))
	Sat Jan 11 00:19:11 2020 - [warning] Connection failed 3 time(s)..
	Sat Jan 11 00:19:12 2020 - [warning] Got error on MySQL connect: 2003 (Can't connect to MySQL server on '192.168.120.13' (111))
	Sat Jan 11 00:19:12 2020 - [warning] Connection failed 4 time(s)..
	Sat Jan 11 00:19:12 2020 - [warning] Master is not reachable from health checker!
	Sat Jan 11 00:19:12 2020 - [warning] Master 192.168.120.13(192.168.120.13:6606) is not reachable!
	Sat Jan 11 00:19:12 2020 - [warning] SSH is reachable.
	``` 
  日志中可以看到，他会向cetus写入一条sql语句,将备用master转成正式。
  ``` bash
    All other slaves should start replication from here. Statement should be: CHANGE MASTER TO MASTER_HOST='192.168.120.14'  MASTER_PORT=6626, MASTER_AUTO_POSITION=1, MASTER_USER='repl', MASTER_PASSWORD='xxx';
  ```
  3. 当原master恢复正常后，可以将它恢复为slave节点  
  #change master to master_host='192.168.120.14',master_user='repl',master_port=6626,master_password="mysql123",master_auto_position = 1;  
  #start slave;  
  4. 在192.168.120.14（当前的master）  
  mysql> show slave hosts;
+-----------+------+------+-----------+--------------------------------------+
| Server_id | Host | Port | Master_id | Slave_UUID                           |
+-----------+------+------+-----------+--------------------------------------+
|      6606 |      | 6606 |      6635 | 34e91deb-32d7-11ea-ae86-000c29350c8b |
|      6636 |      | 6616 |      6635 | 43fd16e7-32d7-11ea-9400-000c29ca619a |
+-----------+------+------+-----------+--------------------------------------+
  5. 在cetus的管理端，去更新主从信息
  ySQL [(none)]> select * from backends；
+-------+-------------+---------------------+---------+------+-----------------+------------+------------+-------------+
| PID   | backend_ndx | address             | state   | type | slave delay(ms) | idle_conns | used_conns | total_conns |
+-------+-------------+---------------------+---------+------+-----------------+------------+------------+-------------+
| 19544 | 1           | 192.168.120.13:6606 | down    | ro   | 0               | 0          | 0          | 0           |
| 19544 | 2           | 192.168.120.15:6616 | unknown | ro   | 0               | 100        | 0          | 100         |
| 19544 | 3           | 192.168.120.14:6626 | up      | rw   | NULL            | 100        | 0          | 100         |
| 19545 | 1           | 192.168.120.13:6606 | down    | ro   | 0               | 0          | 0          | 0           |
| 19545 | 2           | 192.168.120.15:6616 | unknown | ro   | 0               | 100        | 0          | 100         |
| 19545 | 3           | 192.168.120.14:6626 | up      | rw   | NULL            | 100        | 0          | 100         |
+-------+-------------+---------------------+---------+------+-----------------+------------+------------+-------------+
  update backends set state='up' , type='ro' where address='192.168.120.13:6606';
  
  
  
