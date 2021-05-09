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
- 打开[官方地址](https://ftp.postgresql.org/pub/source/)，下载12的源代码（最新版本）
- initdb可以安全的重复调用，若-D后的目录没有权限或者已经存在则会调用失败，该目录可以用环境变量PGDATA代替：
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
- 上面安装时指定的环境变量PGDATA或者-D后的目录叫做数据存储目录，默认情况下，数据存储中目录有三个配置，分别是主服务器配置postgresql.conf，认证配置pg_hba.conf，用户名称映射配置pg_ident.conf
- 数据存储目录和各配置的位置也可以单独指定，比如可以通过环境变量PGDATA或-D只指定主服务配置的位置，并在主服务的配置中包含其他配置（hba_file，ident_file）和数据存储目录的位置（data_directory），当然还有其他方法，可以参考[此处](http://www.postgres.cn/docs/12/runtime-config-file-locations.html)

- 这里简单描述postgresql.conf的设置，详细描述见[文档](http://www.postgres.cn/docs/12/runtime-config-connection.html)：
    ```sh
    # 位置
    data_directory = '数据存储的目录'
    hba_file = '认证配置pg_hba.conf的目录'
    ident_file = '用户名称映射配置pg_ident.conf的目录'
    external_pid_file = '服务器进程的pid文件的目录'

    # 连接
    listen_addresses = '0.0.0.0（IPv4监听地址）'
    port = 5432 # TCP端口
    max_connections = 100 # 最大并发连接数
    tcp_keepalives_idle = 0 # 正常发送TCP keepalive的周期（秒），0为按照操作系统设置
    tcp_keepalives_interval = 0 # 未被客户端确认，重新发送TCP keepalive消息的频率（秒），这之后就判断为断开，0为按照操作系统设置
    tcp_keepalives_count = 0 # 未被客户端确认，重新发送TCP keepalive消息的数量，这之后就判断为断开，0为按照操作系统设置
    tcp_user_timeout = 0 # 指定tcp套接字TCP_USER_TIMEOUT参数（毫秒），意思是在指定时间里未收到分组的ack，就强制关闭连接，0为按照操作系统设置

    # 认证
    authentication_timeout = 60 # 客户端握手超时时间（秒）
    password_encryption = md5 # 对role的密码的加密算法

    # SSL
    ssl = false # 取消ssl认证

    # 内存
    # TODO
    ```

## 简单运维

- 之所以叫简单运维呢，因为适合强度低的项目日常使用和维护。