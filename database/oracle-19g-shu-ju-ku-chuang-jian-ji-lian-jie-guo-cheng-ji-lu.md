# \[Oracle 19g] 数据库创建及连接过程记录

#### docker创建数据库

> \--name oracle-19c // 指定容器名称&#x20;
>
> \-p 1521:1521 // 绑定端口&#x20;
>
> \-e ORACLE\_SID=hk // 指定oracle的sid&#x20;
>
> \-e ORACLE\_PDB=root // 指定pdb的用户名&#x20;
>
> \-e ORACLE\_PWD=123123 // 指定pdb的密码&#x20;
>
> \-v /f/docker/oracle/oradata:/opt/oracle/oradata // 指定路径映射 宿主机文件路径:容器文件路径 zqldocker/oracle:oracle-19cdocker // 指定docker镜像名&#x20;
>
> exec // 命令行&#x20;
>
> \-it // 后台运行&#x20;
>
> /bin/bash // 命令行

```
docker run --name oracle-19c -p 1521:1521 -p 5500:5500 -e ORACLE_SID=hk -e ORACLE_PDB=root -e ORACLE_PWD=123123 -v /f/docker/oracle/oradata:/opt/oracle/oradata zqldocker/oracle:oracle-19cdocker exec -it oracle-19c /bin/bash
```

#### 登录数据库

> 进入到docker容器后执行

```
sqlplus / as sysdba
```

#### 查看文件保存地址

> 用于查看文件保存的位置 例如返回了 /u01/app/oracle/oradata/XE/dba.dbf 等 那么下面创建数据库的路径前缀就是 '/u01/app/oracle/oradata/XE/' 路径最后的'XE'其实就是对应的serviceName

```
select name from v$datafile;
```

#### 创建库

> 下面创建语句可以知道信息
>
> 1. DATAFILE 数据保存文件保存地址
> 2. SIZE 100M 初始化大小
> 3. AUTOEXTEND ON NEXT 32M 自动扩充, 一次扩充32M
> 4. MAXSIZE 500M 最大500M

```
CREATE TABLESPACE admin_db LOGGING DATAFILE '/u01/app/oracle/oradata/XE/admin_db.dbf' SIZE 100M AUTOEXTEND ON NEXT 32M MAXSIZE 500M EXTENT MANAGEMENT LOCAL;
```

#### 创建临时库

```
create temporary tablespace admin_db_temp tempfile '/u01/app/oracle/oradata/XE/admin_db_temp.dbf' size 100m autoextend on next 32m maxsize 500m extent management local;
```

#### 创建用户

```
> 此处创建用户'admin_db', 密码123123, 并指定数据库和临时库
create user admin_db identified by 123123 default tablespace admin_db temporary tablespace admin_db_temp;
```

#### 设置用户权限

> 给admin\_db用户指定dba权限

```
grant dba to admin_db;
```

#### 连接数据库

```
// Spring Boot 2
spring.datasource.url=jdbc:oracle:thin:@127.0.0.1:1521:XE
spring.datasource.username=ADMIN_DB
spring.datasource.password=123123
```
