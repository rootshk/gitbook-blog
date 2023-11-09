# \[MySQL] CentOS 7 安装 MySql 5.7 记录

### 前置准备

```shell
yum update -y
yum upgrade -y
```

### 下载MySQL 5.7安装源

```shell
wget http://repo.mysql.com/mysql57-community-release-el7-11.noarch.rpm
```

### 安装MySQL源

```shell
rpm -Uvh mysql57-community-release-el7-11.noarch.rpm
```

### 导入密钥

```shell
rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
```

### 启动MySQL

```shell
# 启动
systemctl start mysqld.service
# 查看状态
systemctl status mysqld.service
```

#### 打印如下日志

```log
● mysqld.service - MySQL Server
   Loaded: loaded (/usr/lib/systemd/system/mysqld.service; enabled; vendor preset: disabled)
   Active: active (running) since 一 2022-08-22 17:54:55 CST; 26s ago
     Docs: man:mysqld(8)
           http://dev.mysql.com/doc/refman/en/using-systemd.html
  Process: 23800 ExecStart=/usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid $MYSQLD_OPTS (code=exited, status=0/SUCCESS)
  Process: 23750 ExecStartPre=/usr/bin/mysqld_pre_systemd (code=exited, status=0/SUCCESS)
 Main PID: 23804 (mysqld)
   CGroup: /system.slice/mysqld.service
           └─23804 /usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid

8月 22 17:54:52 dev systemd[1]: Starting MySQL Server...
8月 22 17:54:55 dev systemd[1]: Started MySQL Server
```

### 获取初始化密码

```
grep 'temporary password' /var/log/mysqld.log
```

#### 打印：

```
2022-08-22T09:54:53.189408Z 1 [Note] A temporary password is generated for root@localhost: frkKsK_:9Pqk

这里的`frkKsK_:9Pqk`就是密码

```

### 参考

https://cloud.tencent.com/developer/article/1886339
