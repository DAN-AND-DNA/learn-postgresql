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
- 为什么是centos7？ 因为稳定，如果外部环境允许，可以考虑:

    ```sh
    sudo systemctl stop firewalld;
    sudo systemctl disable firewalld;
    sudo setenforce 0;
    sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config;
    ```
- 打开[官方地址](https://ftp.postgresql.org/pub/source/)，下载12的源代码（最新版本）:
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
    /data/pg_build/bin/initdb -D /data/pg_data
    /data/pg_build/bin/pg_ctl -D /data/pg_data -l logfile start

    # 关闭
    sudo su - postgres
    /data/pg_build/bin/pg_ctl -D /data/pg_data stop -m fast
    ```

## systemd

- 现代Linux上的systemd服务可以很方便的帮我们托管postgrsql的主进程和子进程：

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

    ExecStart=/data/pg_build/bin/pg_ctl start -D ${PGDATA} -w
    ExecStop=/data/pg_build/bin/pg_ctl stop -D ${PGDATA} -m fast
    ExecReload=/data/pg_build/bin/pg_ctl reload -D ${PGDATA}
    TimeoutSec=0

    [Install]
    WantedBy=multi-user.target
    EOF
    ```
- 这里还可以参考postgresql的[官方文档](http://postgres.cn/docs/12/server-start.html)，

## archlinux
- 为什么是archlinux？ 因为干净，方便定制：
    ```sh
    #TODO
    ```

## 简单运维

- 之所以叫简单运维呢，因为适合强度低的项目日常使用和维护。