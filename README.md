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
- [概念简介和SQL](#概念简介和SQL)
    - [角色](#角色)
    - [数据库](#数据库)
    - [表空间](#表空间)
    - [类型转换](#类型转换)
    - [常用数据类型](#常用数据类型)
    - [分区](#分区)
    - [索引](#索引)
    - [增删改查](#增删改查)
    - [优化](#优化)
- [数据安全](#数据安全)
- [简单运维](#简单运维)
    - [SQL转储备份](#SQL转储备份)
    - [停机冷备份](#停机冷备份)
    - [基础热备份（可并发）](#基础热备份（可并发）)
    - [基于wal文件复制的后备机](#基于wal文件复制的后备机)
    - [基于wal流复制的后备机](#基于wal流复制的后备机)
- [开发](#开发)
    - [golang](#golang)
- [监控和基准测试](#监控和基准测试)
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
    sudo swapoff -a
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
- 更详细的内容可以参考[文档](http://postgres.cn/docs/12/install-procedure.html)
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
- 视图[pg_settings](http://www.postgres.cn/docs/12/view-pg-settings.html)里有对运行时参数的介绍，通过context列可以知道参数生效策略，这里简单说下：
    ```sh
    internal # 无法直接修改，需要重新初始化数据库
    postmaster # 需重启
    sighup # 需reload
    user # 当前会话
    ```
- 比如，[ALTER SYSTEM](http://www.postgres.cn/docs/12/sql-altersystem.html)和postgresql.conf可以修改全局配置，其中的一部分参数可以在修改后直接通pg_ctl reload来生效，还有一小部分在服务器启动后就不能变更，除非重启服务器比如listen_addresses（监听ip），但大部分是可以实时修改的，其中又分新对话生效和当前对话生效，前者可以用[ALTER DATABASE](http://www.postgres.cn/docs/12/sql-alterdatabase.html)和[ALTER ROLE](http://www.postgres.cn/docs/12/sql-alterrole.html)在设置之后的新的对话里生效，在局部范围内覆盖全局设置，后者可以通过[SET](http://www.postgres.cn/docs/12/sql-set.html)修改当前会话的配置以覆盖全局设置

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
    password_encryption = md5 # 对角色密码的加密算法

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
    log_statement = none # 记录何种SQL 可以是none ddl（数据定义语句） mod（包含ddl和修改操作）all为全部
    
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
## pg_hba.conf
- 这里简单介绍linux下的pg_hba.conf的一些常用设置项，完整内容或不同平台的请见[文档](http://www.postgres.cn/docs/12/auth-pg-hba-conf.html)：
    ```sh
    # postgresql里的role和user基本一样，只是有没有默认的login权限罢了

    #连接类型                       #database                      #user    #address       #auth-method
    local # 匹配unix套接字          all # 匹配全部                  xxx      192.168.1.7    reject
    host # 匹配ssl或普通连接        sameuser # 匹配user同名的数据库  xxx      192.168.1.0/24 trust
    hostssl # 匹配ssl加密的连接     samerole # 匹配role同名的数据库  xxx      0.0.0.0/0      md5      
    hostnossl # 匹配无ssl加密的连接 replication                     xxx      ::0/0          password #明文

    ```
## 概念简介和SQL


## 角色
- 从视图pg_roles里可以查询到很多角色，其中postgres是postgresl初始化后会有唯一一个可登录的角色，，这个角色是超级用户，其他为默认权限相关的角色（组角色）具体说明参考[文档](http://www.postgres.cn/docs/12/default-roles.html)：
    ``` sh
    pg_read_all_settings  # 可以访问所有的配置变量
    pg_read_all_stats     # 可以访问所有的pg_stat_*统计视图 
    pg_stat_scan_tables   # 可以执行监控函数
    pg_monitor            # 可以访问监控视图和函数
    pg_signal_backend     # 可以发信号中断会话的请求或者会话本身
    pg_read_server_files  # 可以读任意文件
    pg_write_server_files # 可以写任意文件
    pg_execute_server_program # 执行程序
    ```

- postgres可以创建其他角色，其中postgresql里的角色和用户是没有区别的，除了用户是默认具有登录权限的，且两者都属于全局范围：
    ```sh
    SELECT * FROM pg_roles; # 查询全部角色
    \du # 查询全部角色 # psql 
    CREATE ROLE xxxx; # 创建角色 （无法登录）
    CREATE ROLE xxxx LOGIN; # 创建角色 （可登录）
    CREATE USER xxxx; # 创建用户 （可登录）
    psql -U xxxx # 登录 # psql
    ```

- 角色需要赋权才能正常使用，可以在创建的时候赋权也可以之后追加：
    ```sh
    CREATE ROLE xxxx SUPERUSER; # 超级用户自动拥有以下权限
    CREATE ROLE xxxx CREATEDB; # 创建数据库
    CREATE ROLE xxxx CREATEROLE; # 创建，修改，删除角色属性
    CREATE ROLE xxxx REPLICATION LOGIN; # 流复制
    CREATE ROLE xxxx PASSWORD 'password' # 登录密码
    ```
- 一个角色上可能绑定有大量对象关系（比如同名数据库），所以删除角色的的时候需要转移这些古旧的对象给其他角色或者直接删除，直接DROP会提示需要转移。建议将业务角色限制在同一个数据库中（不给创建数据库的权限），否则转移的时候会需要在每个有该角色的数据库进行转移：
    ```sh
    REASSIGN OWNERD BY xxxx TO yyyy;  # 每个有该角色的数据库
    DROP OWNER BY xxxx; # 每个有该角色的数据库

    DROP ROLE xxxx; # 全局
    ```
## 数据库
- 从表pg_database里可以查询所有的数据库，初始化成功后会默认创建1个postgres表（如果postgres作为初始用户），2个模板数据库（template0和template1），其中postgres是直接从template1拷贝而来。数据库属于postgresql逻辑中的最高层，单个服务器实例可以管理多个数据库，但应用的每个连接只能单次访问其中一个数据库，所以不同的数据库可以提供给多个不同项目作为物理上的隔离。创建的sql语句可以参考[文档](http://www.postgres.cn/docs/12/sql-createdatabase.html)，删除数据库的时候需要是拥有者或超级用户，而且不能处于要删的数据库里，不然会报错：
    ```sh
    CREATE DATABASE xxx OWNER xxx; # 为角色创建数据库
    DROP DATABASE name; # 删除数据库
    ```
- 可以在pg_hba.conf里配置对角色和数据库的访问进行限制，比如添加如下内容，在内部网络对可信租户的应用进行隔离：
    ```sh
    host    samerole    xxx    192.168.1.0/24      md5      #通过普通的tcp连接访问登录角色同名的数据库
    ```

- 可以在创建数据库的时候指定一个模板数据库，这个模板数据库可以包含你自定义的函数和数据，也可以是默认的template1，但一般不会使用template0作为模板数据库，因为不包含一些必要的配置和数据，原因请参见[文档](http://www.postgres.cn/docs/12/manage-ag-templatedbs.html)：
    ```sh
    CREATE DATABASE xxx OWNER xxx TEMPLATE template1;
    ```

- 可以通过系统视图[pg_settings](http://www.postgres.cn/docs/12/view-pg-settings.html)或SHOW ALL找到会话级别的参数，通过如下sql语句修改在数据库范围的会话配置：
    ```sh
    ALTER DATABASE xxx SET yyy TO off; # 修改
    ALTER DATABASE xxx RESET yyy； # 还原
    ```
    
## 表空间
- 表空间代表实际底层文件系统在postgresql里的映射，可以把postgresql里的对象（数据库，表，索引）单独分配在不同的表空间，完成数据分区间迁移、粗颗粒负载均衡和io性能优化等操作：
    ```sh
    CREATE TABLESPACE yyy LOCATION '/data1/pg_data'; # 创建表空间
    CREATE DATABASE xxx OWNER xxx TABLESPACE yyy; # 分配数据库表空间
    DROP TABLESPACE yyy # 删除表空间
    ```
- 若数据库分配在某个表空间里，那在该数据库里创建的对象默认就会分配在对应的表空间里面，其中包括表，索引，临时表之类
- 详细内容可以参考[CREATE TABLESAPCE](http://www.postgres.cn/docs/12/sql-createtablespace.html)

## 类型转换

-  请见[文档](http://www.postgres.cn/docs/12/sql-expressions.html#SQL-SYNTAX-TYPE-CASTS)

## 常用数据类型

    ```sh
    名字                  存储尺寸        描述	            范围                                        
    smallint	          2字节	        小范围整数          -32768 to +32767
    integer         	  4字节	        整数的典型选择      -2147483648 to +2147483647
    bigint  	          8字节	        大范围整数          -9223372036854775808 to +9223372036854775807
    decimal	              可变	        用户指定精度，精确   最高小数点前131072位，以及小数点后16383位
    numeric	              可变	        用户指定精度，精确   最高小数点前131072位，以及小数点后16383位
    real	              4字节	        可变精度，不精确     6位十进制精度
    double precision	  8字节	        可变精度，不精确     15位十进制精度
    smallserial	          2字节	        自动增加的小整数     1到32767
    serial	              4字节	        自动增加的整数       1到2147483647
    bigserial	          8字节	        自动增长的大整数     1到9223372036854775807
    character varying     有限制的变长                      最大1G
    character             定长，空格填充                    最大1G
    text	              无限变长，被压缩                  
    ```

- smallint（int2），integer（int，int4），bigint（int8）是可用的整数类型，算术运算快
- numeric，deciaml是任意精度数字，实际使用是形如numeric(m, n)，其中m为精度，即总位数，n为标度，即小数点后的位数，可以保证数据精度和计算准确，但是算术运算慢，可以支持字符串NaN（非数字）
- real（float4），double precision（float，float8）是浮点类型，不准确，可以支持字符串Infinity，-Infinity，NaN（非数字）
- character varying（varchar），character（char），text是字符类型，字符类型在postgresql都会被压缩，实际使用是形如character varying(n)来表示上限为n个字符，只存储实际长度的字符串，最大1G。character(n)来表示上限为n个字符，多余位置被空格填充，不填为1个字符串长度，最大1G。text不限制大小，等价于character varying不填n，无大小限制，性能上是character varying和text最快。

```sh
名字	                                存储尺寸	描述
timestamp [ (p) ] [ without time zone ]	8字节	日期+一天中时间（无时区）
timestamp [ (p) ] with time zone	    8字节	日期+一天中时间（有时区）
date	                                4字节	日期（无时间）
time [ (p) ] [ without time zone ]	    8字节	一天中的时间（无日期）
time [ (p) ] with time zone	            12字节	一天中的时间（无日期），带有时区
```
- 上表中日期和时间的类型，timestamp用于表示时间和日期，date只表示日期，time只表示时间，p表示秒后面的小数点位数（0到6），实际使用是形如：
```sh
# timestamp
SELECT timestamp '2021-05-15 22:20:03.3123'; # 2021-05-15 22:20:03.3123
SELECT timestamp(1) '2021-05-15 22:20:03.3123'; # 2021-05-15 22:20:03.3
SELECT timestamp(3) '2021-05-15 22:20:03.3123'; # 2021-05-15 22:20:03.312

# time
SELECT time '22:20:03.3123' #  22:20:03.3123
SELECT time(1) '22:20:03.3123'; # 22:20:03.3

# date
SELECT date '2021-05-15';  # 2021-05-15

```
其他数据结构请参考文档

## 表

## 优化
- [sql优化](http://postgres.cn/docs/12/performance-tips.html)

## 数据安全
- 应用一般需要在内存和磁盘做平衡，即性能和安全之间做平衡，为了能做到对磁盘里的文件进行修改，应用往往会映射文件内容到进程内存中，并在适当的时候请求操作系统把修改过内容强制刷新到磁盘中，当然也可以交由操作系统的缓冲，由操作系统决定在适当的时候写回磁盘，上面这么做的好处在于减少没必要的磁盘io。当然现代磁盘本身为了提高磁盘的读写性能也会提供一个缓存（一般是关闭的）。
- 上面这些重复缓存提高了性能，但也提高了保证数据安全的复杂度，所以一些应用会自己管理文件缓冲，在适当的时候强制操作系统刷盘，并通过关闭磁盘驱动器的缓存进一步提高数据安全。
- 但是刷盘之间的时延不能够解决数据不丢失的问题，因为不会为了每次修改就刷新全部数据（页为单位，应用的页和操作系统的页不太一样）到磁盘，所以数据库往往会通过提供一个叫做预写日志的功能，一般同步提交事务模式是在数据库应用事务前记录该事务到磁盘，如果是异步事务提交模式，是不等数据落盘直接返回的，因为数据库的后台进程会定时定量的刷还未在磁盘的日志到磁盘
- postgresql里存在一个检查点的概念，那个时候是把内存里的数据强制刷新到磁盘，成功后这之前的很多日志就变得无用并被清理掉，当某次意外断电发生，postgresql里还有未写盘的数据就真的丢失了，这个时候就可以根据日志进行重放恢复断电前的状态，这部分的理解可以参考[wal](http://www.postgres.cn/docs/12/wal.html)和计算机操作系统，wal就是postgresql的预写日志

## 简单运维

## SQL转储备份
- postgresql的大版本升级往往需要使用自带工具来迁移数据，相对比较复杂，但是如果使用SQL转储，可以减少很多复杂的操作，[SQL转储](http://www.postgres.cn/docs/12/app-pgdump.html)可以使用pg_dump备份一个正在运行的数据库而不阻塞其他用户访问数据库，需要注意的是postgresql里的很多工具默认用的角色名都是系统用户名，可以用-U来指定角色名，并指定权限范围内的数据库名（超级用户可以无视权限校验）：
    ```sh
    pg_dump -U xxx xxx > dumpfile
    ```
- 从上面的命令得到的dumpfile（实际上是通过把标准输出重定向到某个文件名）就是个都是普通的文本，里面的内容主要是当前数据库里表和数据的一个快照，但不涉及数据库和表空间的创建和指定，如果需要全部的对象，或者有一些定制选择可以考虑[pg_dumpall](http://www.postgres.cn/docs/12/app-pg-dumpall.html) ，所以pg_dump产生的转储在恢复上面就变得宽松，可以通过如下方式恢复：
    ```sh
    CREATE DATABASE xxx OWNER xxx TEMPLATE template0; # 新建
    psql -U xxx --single-transaction --set ON_ERROR_STOP=on xxx < dumpfile # 导入
    ```
 - 因为转储的内容是对比模板数据库template0的差别而产生的，并且不涉及数据库和表空间的创建，所以还原的起点应该是依据template0新创建一个数据库，然后给这个库导入数据，详细过程可以参考[文档](http://www.postgres.cn/docs/12/backup-dump.html)

 ## 停机冷备份
 - 关停服务器，tar整个目录，从源服务器拷贝到目标服务器或者直接使用rsync和scp等系统工具，文件系统也提供[快照功能](http://www.postgres.cn/docs/12/backup-file.html)

 ## 基础热备份（可并发）
 - 如果观察过postgresql的pg_wal目录，就会发现里面的日志文件名字经常做变更，其实是被回收重用了，上面数据安全章节里有说过，wal日志可以帮助我们从崩溃中恢复，如果想在每次回收前保留这些wal文件可以这样：
    ```sh
    # postgresql.conf

    wal_level = replica # 日志模式
    archive_mode = on  # 打开归档
    archive_command = 'test ! -f /data/pg_arch/%f && cp %p /data/pg_arch/%f' # 本地拷贝
    ```

- 配置了上述参数需要重启服务器，如果archive_command里的shell命令失败会一直重试，直到shell结果的$?环境变量为0，并保留pg_wal里日志不删除，如果archive_command为空字符串，亦不删除pg_wal里的日志（可以考虑在业务不繁忙的自己清理）
- 有了上述不断被归档的日志文件，还需要做一个基础备份作为未来恢复时的起点，所谓基础备份就是当前数据库的一个快照， 打开一个新连接输入，但不要关闭该连接：
    ```sh
    SELECT pg_start_backup('label', false, false); # 这个命令的意思是启动一次备份，label是识别本次备份的标签
    ```
- 等上面的命令返回后就可以把整个数据存储目录用tar打包起来，这个命令实际上发起了一次[检查点](http://postgres.cn/docs/12/wal-configuration.html)来把内存里数据都刷到磁盘，还创建备份期的新wal，检查点的位置实际就写在这些新wal中
- 等上面的tar命令结束就可以在刚才的连接输入如下内容：
    ```sh
    SELECT * FROM pg_stop_backup(false);
    ```
- 上面的命令会返回备份期wal的信息，并在备份结束的时候，在pg_wal目录里添加了一个backup文件表示备份期间wal清单，包括开始的wal位置、结束的wal位置和检查点的位置，并对这些备份期间的新wal进行了归档，最后切换到新的wal，上面就完成了基础备份
- 下面在其他地方恢复，对基础备份进行解压并在目录里创建recovery.signal：
    ```sh
    # postgresql.conf

    wal_level = replica # 日志模式 （可选）
    archive_mode = off  # 打开归档 （可选）
    archive_command = '' # 本地拷贝 （可选）
    restore_command = 'cp /data/pg_arch/%f "%p"' #
    ```
- 启动服务器，一般来说会正常启动，如果发生问题请参考[这里](http://www.postgres.cn/docs/12/continuous-archiving.html)

## 基于wal文件复制的后备机
- 基础热备份过程同上文，对基础备份解压并创建standby.signal文件，差别就是wal归档的目录，可以考虑让主服务器发送归档到后备机的某个目录上，比如下文的（/data/remote_pg_arch），后备的配置如下：
    ```sh
    # postgresql.conf

    wal_level = replica # 日志模式 （可选）
    archive_mode = off  # 打开归档 （可选）
    archive_command = '' # 本地拷贝 （可选）
    restore_command = 'cp /dataremote_pg_arch/%f %p'
    archive_cleanup_command = 'pg_archivecleanup /data/remote_pg_arch %r' # 清理无用wal
    ```
- 基于wal文件复制的后备机，对数据库的消耗非常小，速度自然很优秀，但容易因为归档的延迟导致后备机在主服务器奔溃时落后，届时主备切换需手动干预，保证数据进度一致

## 基于wal流复制的后备机
- 流复制默认是异步的，需要在主服务器配置wal_keep_segments防止wal不被过早清理和重用避免导致后备失败，如果有必要需要拷贝主服务器的wal归档到后备服务器，主服务器操作和配置如下：
    ```sh
    CREATE ROLE xxxx REPLICATION LOGIN PASSWORD 'yyyy'; # 用户需要流复制权限

    # pg_hba.conf

    # TYPE  DATABASE        USER            ADDRESS                 METHOD
    host    replication     xxxx            192.168.1.0/24          md5

    # postgresql.conf

    wal_keep_segments = 20
    ```

- 如果需要主服务器进行同步复制可以参考[这里](http://www.postgres.cn/docs/12/warm-standby.html#SYNCHRONOUS-REPLICATION)进行配置：
    ```sh
    # postgresql.conf

    synchronous_standby_names = 'hostname' # 备库的host或者ip
    synchronous_commit = 'remote_write'
    ```

- 基础热备份过程同上文，对基础备份解压并创建standby.signal文件，备库的配置如下：
    ```sh
    # postgresql.conf

    primary_conninfo = 'host=192.168.1.37 port=5432 user=xxxx password=yyyy' # 假设主服务器在192.168.1.37
    wal_level = replica # 日志模式 （可选）
    archive_mode = off  # 打开归档 （可选）
    archive_command = '' # 本地拷贝 （可选）
    ```

## 监控和基准测试
- [统计](http://postgres.cn/docs/12/monitoring-stats.html)
- [锁](http://postgres.cn/docs/12/view-pg-locks.html)
- [pg_bench](http://postgres.cn/docs/12/pgbench.html)
- [pg_stat_statements](http://postgres.cn/docs/12/pgstatstatements.html)