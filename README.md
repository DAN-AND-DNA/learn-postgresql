# 实战postgresql 12


## 内容
- [为什么是postgresql](#为什么是postgresql)
- [源码安装](#源码安装)
    - [centos7](#centos7)
    - [systemd](#systemd)
    - [archlinux](#archlinux)
- [配置详解](#配置详解)
    - [pg_hba.conf](#pg_hbd.conf)
- [命令](#命令)
- [sql](#sql)
    - [角色](#角色)
    - [数据库](#数据库)
- [简单运维](#简单运维)
    - [冷备份](#冷备份)
    - [热备份](#热备份)
    - [raid10](#raid10)
    - [物理机多租户](#物理机多租户)
    - [私有云多租户](#私有云多租户)
    - [多磁盘](#多磁盘)
    - [同步主从](#同步主从)
    - [异步主从](#异步主从)
- [开发](#开发)
    - [k8s](#k8s)
    - [golang](#golang)
    - [c++](#c++)
    - [lua](#lua)
    - [rust](#rust)
- [监控](#监控)
    - [观测](#观测)
    - [统计](#统计)
- [内核参数](#内核参数)

## 为什么是postgresql

- 因为简单
## 源码安装

- 为什么是源代码安装？ 理由很多，比如因为不难安装、追求稳定、版本可控、源代码审核、理解底层原理、定制和扩展等。
## centos7
- postgresql需要操作系统提供进程间通信，如果是Linux，默认会使用POSIX信号量，不需要特别设置内核参数，如果是其他系统可能需要配置，可以参考[官方文档](http://www.postgres.cn/docs/12/kernel-resources.html#SYSVIPC)
- 如果外部环境允许，可以考虑：

    ```sh
    sudo systemctl stop firewalld;
    sudo systemctl disable firewalld;
    sudo setenforce 0;
    sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config;
    ```
- 打开[官方地址](https://ftp.postgresql.org/pub/source/)，下载12的源代码（最新版本）：
    ```sh
    # 安装
    sudo yum install -y zlib-devel.x86_64 readline-devel.x86_64 gcc.x86_64
    tar -zxvf postgresql-12.*.tar.gz
    sudo mkdir -p /data/pg_data /data/pg_build
    cd postgresql-12.*
    ./configure --prefix=/data/pg_build
    make -j4
    sudo make install
    sudo adduser postgres
    chown postgres /data/pg_data

    # 运行
    sudo su - postgres
    /data/pg_build/bin/pg_ctl -D /data/pg_data initdb
    /data/pg_build/bin/pg_ctl -D /data/pg_data -l logfile start

    # 关闭
    sudo su - postgres
    /data/pg_build/bin/pg_ctl -D /data/pg_data stop -m fast
    ```
- 上面的initdb可以安全的重复调用，若目录已经存在则会调用失败，该目录可以用环境变量PGDATA代替
## systemd

- 现代Linux上的systemd服务可以很方便的帮我们托管postgresql的主进程和子进程，并通过添加[环境变量](http://www.postgres.cn/docs/12/kernel-resources.html#LINUX-MEMORY-OVERCOMMIT)来改变子进程的行为：

    ```sh
    cat << EOF | tee /etc/systemd/system/postgresql.service
    [Unit]
    Description=PostgreSQL database server
    Documentation=man:postgres(1)
    After=network.target

    [Service]
    Type=forking
    User=postgres
    OOMScoreAdjust=-1000
    Environment=PG_OOM_ADJUST_FILE=/proc/self/oom_score_adj
    Environment=PG_OOM_ADJUST_VALUE=0
    Environment=PGDATA=/data/pg_data

    ExecStart=/data/pg_build/bin/pg_ctl start -D \${PGDATA} -w
    ExecStop=/data/pg_build/bin/pg_ctl stop -D \${PGDATA} -m fast
    ExecReload=/data/pg_build/bin/pg_ctl reload -D \${PGDATA}
    TimeoutSec=0

    [Install]
    WantedBy=multi-user.target
    EOF
    ```
- 这里还可以参考postgresql的[文档](http://postgres.cn/docs/12/server-start.html)，

## archlinux
- 为什么是archlinux？ 因为方便定制：
    ```sh
    # TODO
    ```
## 配置详解
- 上面安装时指定的环境变量PGDATA或者-D后的目录叫做数据存储目录，默认情况下，数据存储目录中有三个配置，分别是主服务器配置postgresql.conf，认证配置pg_hba.conf，用户名称映射配置pg_ident.conf
- 实际上，数据存储目录和各配置的位置可以单独指定，比如可以通过环境变量PGDATA或-D来指定主服务配置的位置，并在主服务配置中包含其他配置（hba_file，ident_file）和数据存储目录的位置（data_directory），当然还有其他方法，可以参考[此处](http://www.postgres.cn/docs/12/runtime-config-file-locations.html)

- 这里简单介绍linux下的postgresql.conf的一些常用设置项，完整内容或不同平台的请见[文档](http://www.postgres.cn/docs/12/runtime-config-connection.html)：
    ```sh
    # 1.位置
    data_directory = '数据存储的目录'
    hba_file = '认证配置pg_hba.conf的位置'
    ident_file = '用户名称映射配置pg_ident.conf的位置'
    external_pid_file = '服务器进程的pid文件的位置'

    # 2.连接
    listen_addresses = '0.0.0.0（IPv4监听地址）'
    port = 5432 # TCP端口
    max_connections = 100 # 最大并发连接数
    tcp_keepalives_idle = 0 # 正常发送TCP keepalive消息的周期（秒），0为按照操作系统设置
    tcp_keepalives_interval = 0 # 未被客户端确认，重新发送TCP keepalive消息的频率（秒），在这之后就判断为断开，0为按照操作系统设置
    tcp_keepalives_count = 0 # 未被客户端确认，重新发送TCP keepalive消息的数量，在这之后就判断为断开，0为按照操作系统设置
    tcp_user_timeout = 0 # 指定tcp套接字的TCP_USER_TIMEOUT参数（毫秒），该参数的意思是在指定时间里未收到发送分组的ack，就强制关闭连接，0为按照操作系统设置

    # 3.认证
    authentication_timeout = 60 # 客户端握手超时时间（秒）
    password_encryption = md5 # 对role的密码的加密算法

    # 4.SSL
    ssl = off # 取消ssl认证

    # 5.内存
    shared_buffers = 128MB # 共享内存缓冲区大小，属于全局共享
    huge_page = try # 对共享内存开启标准巨型页，需要操作系统支持，并配置系统内核中的hugepages相关参数来调整巨型页池行为
    temp_buffes = 8MB # 会话层级的临时表缓冲区
    work_mem = 4MB # 操作层级的可用内存，属于进程本地内存，若实际内存大于该值，就会写入临时结果到磁盘中的临时文件中
    max_stack_depth = 2MB # 可以根据实际系统的栈深度来定，比如 ulimit -s 再减去1-2MB的大小
    shared_memory_type = mmap # 共享内存在操作系统上的实现方式，其中mmap是比较常见的方式，详细可以参考操作系统api
    dynamic_shared_memory_type = posix # 动态共享内存在操作系统上的实现方式，若是posix，会调用sham_open来生成基于内存的临时目录，该目录供mmap映射到进程中进行使用，结束使用时被系统清理

    # 6.磁盘
    temp_file_limit = -1 # 磁盘中的临时文件大小，-1即没有限制

    # 7.内核
    max_files_per_process = 1000 # postgresql进程的文件描述符上限，这个选项要依赖操作系统的设置
    
    # 8.减低VACUUM和ANALYZE命令的io开销
    vacuum_cost_delay = 0 # 大于0表示开启，并作为VACUUM和ANALYZE命令执行进程的休眠时间（毫秒），休眠结束重新计数
    vaccum_cost_page_hit = 1 # 在共享缓存中找到要清理的数据所在页的代价
    vaccum_cost_page_miss = 10 # 要清理的数据不在共享缓存的页中，从磁盘内加载到共享缓存的代价
    vaccum_cost_page_dirty = 20 # 清理的数据需要修改数据导致脏页，并需刷回磁盘的代价
    vaccum_cost_limit = 200 # 根据累计的代价值进行休眠vacuum_cost_delay时间

    # 9.后台写入进程
    bgwriter_delay = 200ms # 刷脏页到磁盘的周期
    bgwriter_lru_maxpages = 100 # 最大刷脏页数量
    bgwriter_lru_multipages = 2.0 
    bgwriter_flush_after = 0 # 0为按照操作系统的配置来，2MB是最大限制，这个选项是后台写入数据量达到一定量就强制操作系统将高速缓冲的脏页刷到磁盘

    # 10.异步
    effective_io_concurrency = 1 # 全局范围的对磁盘的并发io数，最好在创建表空间时覆盖该参数，过高的并发数也容易导致磁盘进行排队，延迟过高
    max_worker_processes = 8 # worker进程池的大小
    max_parallel_workers = 8 # 每个操作支持的最大并行worker数，worker代表工作进程
    max_parallel_workers_per_gather = 2 # 当postgreql查询优化器判断需要并行查询时，会创建gather节点，并决定其worker的数量
    max_parallel_maintenance_workers = 2 # 目前只会影响创建索引的并行worker数
    backend_flush_after = 0 # 0为按照操作系统来，2MB是最大上限，这个选项是当后台写入数据量达到一定量就强制操作系统将高速缓冲的脏页刷到磁盘

    # 11.预写日志
    wal_level = replica  # 不同的wal日志等级，写入内容会有所不同，做热备份建议replica以上，每个等级都会有其短板，比如replica物理备份，会因为主备速度差异，导致备库失败，需要配置其他参数，logical重放备份又可能有冲突的可能
    fsync = on # 是否对wal的写入需要更新到磁盘，但何时真的更新对应的文件系统的元数据和数据本身这个需要设置wal_sync_method
    synchronous_commit = on # 事务提交是否需要等待wal写入磁盘或者已经同步给备库
    wal_sync_method = fsync # 可以根据操作系统提供的API作为参考，比如linux上fsync要阻塞在同步文件系统元数据和数据本身，而fdatasync只在必要时更新文件系统元数据，虽然两者都能绕过操作系统的高速缓冲，但后者少了一些io操作自然快。如果有必要还需要强制禁用磁盘驱动器的写缓存，但是性能就不行了，所以需要在安全和性能做出平衡，详细可以看文档和了解文件系统如何保证写入安全性
    
    

    ```

## 简单运维

- 之所以叫简单运维呢，因为适合强度低的项目日常使用和维护。