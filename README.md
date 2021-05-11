# 实战postgresql 12


## 内容
- [为什么是postgresql](#为什么是postgresql)
- [源码安装](#源码安装)
    - [centos7](#centos7)
    - [systemd](#systemd)
    - [archlinux](#archlinux)
- [配置](#配置)
    - [postgresql.conf](#postgresql.conf)
    - [pg_hba.conf](#pg_hba.conf)
- [命令](#命令)
- [概念简介和SQL](#概念简介和SQL)
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
## 配置
- 上面安装时指定的环境变量PGDATA或者-D后的目录叫做数据存储目录，默认情况下，数据存储目录中有三个配置，分别是主服务器配置postgresql.conf，认证配置pg_hba.conf，用户名称映射配置pg_ident.conf
- 实际上，数据存储目录和各配置的位置可以单独指定，比如可以通过环境变量PGDATA或-D来指定主服务配置的位置，并在主服务配置中包含其他配置（hba_file，ident_file）和数据存储目录的位置（data_directory），当然还有其他方法，可以参考[此处](http://www.postgres.cn/docs/12/runtime-config-file-locations.html)

## postgresql.conf
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
    shared_buffers = 128MB # postgresql的共享内存缓冲区大小，以数据块作为单位读写，属于全局共享，与操作系统的高速缓冲有所重复，所以这个值不用很大，尽量使用操作系统的高速缓冲
    huge_page = try # 对共享缓冲开启标准巨型页，需要操作系统支持，并配置系统内核中的hugepages相关参数来调整巨型页池行为
    temp_buffers = 8MB # 会话层级的临时表的缓冲区，属于进程本地内存
    work_mem = 4MB # 操作层级的内存上限，属于进程本地内存，若实际内存大于该值，就会写入临时结果到磁盘中
    max_stack_depth = 2MB # 可以根据操作系统的实际栈深度来定，比如 ulimit -s 再减去1-2MB的大小
    shared_memory_type = mmap # 共享内存在操作系统上的实现方式，其中mmap是比较常见的方式，详细可以参考操作系统api
    dynamic_shared_memory_type = posix # 动态共享内存在操作系统上的实现方式，若是posix，会调用sham_open来生成基于内存的临时目录，该目录供mmap映射到进程中进行使用，结束使用时被系统清理

    # 6.磁盘
    temp_file_limit = -1 # 每个进程存放临时文件的磁盘大小上限，-1即没有限制

    # 7.内核
    max_files_per_process = 1000 # postgresql进程的文件描述符上限，这个选项要依赖操作系统的设置
    
    # 8.减低VACUUM和ANALYZE命令的io开销
    vacuum_cost_delay = 0 # 大于0表示开启，并作为VACUUM和ANALYZE命令执行进程的休眠时间（毫秒），休眠结束重新计数
    vaccum_cost_page_hit = 1 # 在共享缓冲中找到要清理的数据的代价
    vaccum_cost_page_miss = 10 # 要清理的数据不在共享缓冲中，从磁盘内加载到共享缓冲的代价
    vaccum_cost_page_dirty = 20 # 要清理的数据需要修改数据导致需刷回磁盘的代价
    vaccum_cost_limit = 200 # 根据累计的代价值进行休眠vacuum_cost_delay时间

    # 9.后台写入进程
    bgwriter_delay = 200ms # 刷postgresql中共享缓冲内的脏页数据到磁盘的周期，这里脏页是在postgresql的共享缓存中定义的
    bgwriter_lru_maxpages = 100 # 每次刷磁盘的最大刷脏页数量，这里脏页是在postgresql的共享缓存中定义的
    bgwriter_lru_multipages = 2.0 
    bgwriter_flush_after = 0 # 0为按照操作系统的配置来，2MB是最大限制，这个选项是后台写入数据量达到一定量就强制操作系统将postgresql的共享高速缓冲里的脏页刷到磁盘，这里脏页是在postgresql的共享缓存中定义的，而非操作系统自己的高速缓冲

    # 10.异步
    effective_io_concurrency = 1 # 全局范围的对磁盘的并发io数，最好在创建表空间时覆盖该参数，过高的并发数也容易导致磁盘进行排队，延迟过高
    max_worker_processes = 8 # worker进程池的大小
    max_parallel_workers = 8 # 每个操作支持的最大并行worker数，worker代表工作进程
    max_parallel_workers_per_gather = 2 # 当postgreql查询优化器判断需要并行查询时，会创建gather节点，并决定其worker的数量
    max_parallel_maintenance_workers = 2 # 目前只会影响创建索引的并行worker数
    backend_flush_after = 0 # 0为按照操作系统来，2MB是最大限制，这个选项是后台写入数据量达到一定量就强制操作系统将postgresql的共享高速缓冲里的脏页刷到磁盘，这里脏页是在postgresql的共享缓存中定义的，而非操作系统自己的高速缓冲

    # 11.预写日志 wal（行级数据）
    wal_level = replica  # 不同的wal日志等级，写入内容会有所不同，做热备份建议replica以上，每个等级都会有其短板，比如replica物理备份，会因为主备速度差异，导致备库失败，需要配置其他参数，logical重放备份又可能有冲突的可能
    fsync = on # 是否对wal的写入需要更新到磁盘，但何时真的更新对应的文件系统的元数据和数据本身这个需要设置wal_sync_method
    synchronous_commit = on # 事务提交是否需要等待wal写入磁盘或者已经同步给备库
    wal_sync_method = fsync # 可以根据操作系统提供的API作为参考，比如linux上fsync要阻塞在同步文件系统元数据和数据本身，而fdatasync只在必要时更新文件系统元数据，虽然两者都能绕过操作系统的高速缓冲，但后者少了一些io操作自然快。如果有必要还需要强制禁用磁盘驱动器的写缓存，但是性能就不行了，所以需要在安全和性能做出平衡，详细可以看文档和了解文件系统如何保证写入安全性
    full_page_writes = on # 检查点之后就把每个被修改过的完整的页数据（postgresql定义的）都写到wal里，这样就有了起始点，方便完整的恢复，避免只有行级变更带来多次污染
    wall_compression = off # 日志压缩
    wal_buffers = -1 # wal日志在共享缓冲里的大小 -1表示自动选
    wal_writer_delay = 200ms # wal刷到磁盘的周期（毫秒），否则写入共享缓冲
    wal_writer_flush_after = 1MB # 到达指定大小就写入磁盘，否则写入共享缓冲，设置为0就会立刻刷到磁盘
    commit_delay = 0 # 提交延迟时间内允多事务同时通过一个wal刷到磁盘, 0为立刻写入磁盘
    commit_siblings = 5 # 提交延迟里的最大事务数

    # 12.检查点
    checkpoint_timeout = 5min # wal日志检查点的间隔时间，到了检查点就会根据wal大小删除wal或者重用
    checkpoint_flush_after = 256kB # 到达wal检查点时，需要把postgresql的共享缓存里的全部脏页都刷到磁盘，并在wal上标记该检查点，如果这段时间写入的数据超过配置的大小就让操作系统提前刷postgresql的共享缓冲里的脏页到磁盘，避免之后io峰值过高，大小为0-2mb
    max_wal_size = 1GB # 检查点间隔内的wal最大值，但到达未必删
    min_wal_size = 80MB # 检查点间隔内的wal最小值，小于该值就回收

    # 13.复制wal到后备服务器
    max_wal_senders = 10 # 最大wal发送进程数
    max_keep_segments = 0 # 保留的wal数量，可以保证备库不容易因为主日志被覆盖而导致同步失败，总大小可以通过wal的大小评估
    wal_sender_timeout = 60 # 发送超时，直接中断后备服务，当然同步和异步最好配置不同的大小
    
    # 14.主服务器
    synchronouse_standby_names = 'hostname1,hostname2' # 同步备库的域名和更新策略，具体配置请参考文档，还有一个参数synchronous_commit表示事务同步的形式，其中选项remote_write代表备库写入内存就返回成功，然后主库等待提交的事务就算提交成功了
    vaccum_defer_cleanup_age = 0 # 该参数代表要推迟清理的事务数量，推迟主库上事务的清理可以避免备库因为更新延迟的问题导致查询备库时发生冲突

    # 15.后备服务器（针对后备服务器）
    primary_conninfo = '主库的角色名 主库的角色口令 ssl相关选项 host=主服务器的hostname或ip port=主服务器的端口' # 后备服务器和主服务器的握手信息
    hot_standy = on # 备库是否提供只读服务
    max_standy_streaming_delay = 30s # 备库流复制wal时，因为同步延迟导致查询发生冲突，所以需要对备库的的每个查询进行一定延迟，这个时间代表所有查询的的延迟时间，否则取消该查询
    wal_receiver_status_interval = 10s # 发送wal复制情况给主服务器的周期
    wal_receiver_timeout = 60s  # 后备服务器判断主服务器超时时间
    wal_retrieve_retry_interval = 5s # 重试尝试获取wal的周期

    # 16.查询优化
    TODO

    # 17.错误日志
    log_destination = 'stderr' # 日志目的地 可以同时选stderr csvlog syslog eventlog(windows)
    logging_collector = off # 日志是否落地
    log_directory = 'log' # 日志落地目录，可以是相对目录（数据存储目录），也可以是绝对路径
    log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log' # 日志命名
    log_rotation_age = 1d # 日志轮转周期 0为禁止
    log_rotation_size = 10MB # 日志轮转大小 0为禁止
    log_min_messages = warning # 日志错误等级
    log_min_error_statement = error # 记录SQL语句的错误等级
    log_min_duration_statement = -1 # 记录达到指定时间的SQL语句，可以检出慢查询， -1代表关闭
    debug_print_parse = off # 打印调试信息
    debug_print_rewritten = off # 打印调试信息
    debug_print_plan = off # 打印调试信息
    log_checkpoints = off # 记录检查点和重启点以及一些时间统计
    log_connections = off # 记录连接信息
    log_disconnections = off # 记录断开连接信息
    log_duration = off # 记录语句完成时间
    log_line_prefix = '%m [%p] ' # 每个日志行开头包含的内容，详细内容参考文档
    log_lock_waits = off # 会话等待锁已超过deadlock_timeout，是否要产生一条日志
    log_statement = none # 记录何种SQL ddl为数据定义 mod为ddl all为全部
    
    # 18.运行时统计 （通过pg_stat_*视图）
    track_activities = on # 打开对每个会话的统计
    track_activity_query_size = 1024 # pg_stat_activity.query域的内存大小
    track_counts = on # 打开对每个数据库的统计
    stats_temp_directory = 'pg_stat_tmp' # 统计的临时目录 可以是相对目录（数据存储目录），也可以是绝对路径
    log_statement_stats = off # 对语句进行统计

    # 19.自动清理 (表的选项覆盖可以其中一些选项，请参考文档)
    autovacuum = on # 还需开启 track_counts选项
    log_autovacuum_min_duration = -1 # 记录清理时间达到指定时间 -1为禁用
    # TOOD

    # 20.客户端连接
    client_min_messages = notice # 发给客户端的消息等级
    statement_timeout = 0 # SQL执行超时
    lock_timeout = 0 # 等锁超时

    # 21.锁
    deadlock_timeout = 1s # 超过该时间就开始检查死锁
    max_locks_per_transaction = 64 # 每个事务的最大对象锁
    ```
## pg_hbd.conf
- 这里简单介绍linux下的pg_hba.conf的一些常用设置项，完整内容或不同平台的请见[文档](http://www.postgres.cn/docs/12/auth-pg-hba-conf.html)：
    ```sh
    # postgresql里的role和user基本一样，只是有没有默认的login权限罢了

    #连接类型                       #database                      #user    #address       #auth-method
    local # 匹配unix套接字          all # 匹配全部                           192.168.1.7    reject
    host # 匹配ssl或普通连接        sameuser # 匹配user同名的数据库           192.168.1.0/24 trust
    hostssl # 匹配ssl加密的连接     samerole # 匹配role同名的数据库           0.0.0.0/0      md5      
    hostnossl # 匹配无ssl加密的连接                                          ::0/0          password #明文

    ```
## 概念简介和SQL

## 角色
- postgresl初始化后会有唯一一个可登录的角色，其他为权限相关的角色，即postgres，这个角色是超级用户，该角色可以创建其他角色，其中postgresql里的角色和用户是没有区别的，除了用户是默认具有登录权限的，且两者都属于全局范围：
    ```sh
    SELECT * FROM pg_roles; # 查询全部角色
    \du # 查询全部角色 # psql 
    CREATE ROLE xxxx; # 创建角色 （无法登录）
    CREATE ROLE xxxx LOGIN; # 创建角色 （可登录）
    CREATE USER xxxx; # 创建用户 （可登录）
    psql -U xxxx # 登录 # psql
    ```

- 角色需要赋权才能正常使用，可以在创建的时候加也可以之后追加：
    ```sh
    CREATE ROLE xxxx SUPERUSER; # 超级用户自动拥有以下权限
    CREATE ROLE xxxx CREATEDB; # 创建数据库
    CREATE ROLE xxxx CREATEROLE; # 创建，修改，删除角色属性
    CREATE ROLE xxxx REPLICATION LOGIN; # 流复制
    CREATE ROLE xxxx PASSWORD 'password' # 登录密码
    ```
- 在角色上可能会产生大量对象关系（比如创建了表），所以删除角色的的时候需要转移这些古旧的对象给其他角色或者直接删除，直接DROP会提示需要转移。建议将业务角色限制在同一个数据库中（不给创建数据库的权限），否则转移的时候会需要在每个有该角色的数据库进行转移：
    ```sh
    REASSIGN OWNERD BY xxxx TO yyyy;  # 每个有该角色的数据库
    DROP OWNER BY xxxx; # 每个有该角色的数据库

    DROP ROLE xxxx; # 全局
    ```

## 组
    TODO



## 简单运维

- 之所以叫简单运维呢，因为适合强度低的项目日常使用和维护。