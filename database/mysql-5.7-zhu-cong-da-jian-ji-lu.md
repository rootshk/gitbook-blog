# \[MySQL 5.7] 主从搭建记录

\*\*使用CentOS 7 \*\*

```
$ docker pull centos:centos7
$ docker run -itd --name centos-t1 --net=host centos:centos7
$ docker exec -it centos-t1
centos-t1$ yum update -y
centos-t1$ yum upgrade -y
centos-t1$ yum install vim wget -y

...
```
