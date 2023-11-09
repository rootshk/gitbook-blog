# \[Docker 初始化环境] Mysql 8 + Redis 6

**新电脑到了, 本地环境部署**

## MySQL

### my.cnf文件

```
[mysqld]
hentication_plugin
skip-host-cache
skip-name-resolve
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
secure-file-priv=/var/lib/mysql-files
user=mysql
pid-file=/var/run/mysqld/mysqld.pid
```

### MySQL8 X86版本

```
$ docker pull mysql
$ docker run -itd -p 3306:3306 -e MYSQL_ROOT_PASSWORD=root --name mysql-latest mysql
```

### MySQL8 ARM M1版本

```
$ cd ~/
$ mkdir mysql
$ mkdir mysql/data
$ docker pull mysql/mysql-server:latest
$ docker run -itd --name mysql -p 3306:3306 -v /Users/roothk/mysql/data:/var/lib/mysql -v /Users/roothk/mysql/my.cnf:/etc/my.cnf -e MYSQL_ROOT_PASSWORD=123456 mysql/mysql-server --lower_case_table_names=1
```

### 初始化配置

```
$ docker exec -it mysql /bash/sh

bash-4.2# mysql -u root -p 123456
mysql>CREATE USER 'root'@'%' IDENTIFIED BY 'root';
mysql>GRANT ALL ON *.* TO 'root'@'%';
mysql> flush privileges;
mysql> ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
mysql> flush privileges;
```

## Redis 6

```
$ docker pull redis
$ docker run -itd -p 6379:6379 --name docker-redis redis
```
