# \[Jmeter 压测] 记录jmeter server模式压测

**基于CentOS 7, Jdk 11**

```
server模式压测 需要用到至少三台linux机器
服务器A：部署了被压测程序（jmeter配置界面设置的server host）
服务器B：jmeter-server（实际发起请求的slave端）
服务器C：jmeter（用于向各个jmeter-server发起调用命令的master端）
```

## 在B和C服务器上下载Jmeter

### 下载

#### [最新Jmeter](https://jmeter.apache.org/download\_jmeter.cgi) / [5.4.3版本](https://dlcdn.apache.org/jmeter/binaries/apache-jmeter-5.4.3.tgz)

> $ wget https://dlcdn.apache.org//jmeter/binaries/apache-jmeter-5.4.3.tgz

### 解压

> $ tar -zxvf apache-jmeter-5.4.3.tgz

## 服务器B 配置Jmeter

```
$ cd apache-jmeter-5.4.3
$ vim bin/jmeter.properties

# 关闭ssl连接
server.rmi.ssl.disable=true
```

## 服务器C 配置Jmeter

```
$ cd apache-jmeter-5.4.3
$ vim bin/jmeter.properties

# 关闭ssl连接
server.rmi.ssl.disable=true
# 服务器B的地址，多个以,分割 如果在同一个服务器上就加上设置的端口port（IP:PORT）
remote_hosts=172.1.0.1,172.1.0.2,172.1.0.3,172.1.0.4
```

## 启动服务器B（可多个服务器）

```
# 执行后就可以挂着等待命令了
$ sh ./bin/jmeter-server
```

## 启动服务器C

> 将GUI端配置好的demo.jmx保存到服务器C的apache-jmeter-5.4.3目录下

### 初始化

```
$ mkdir report
$ vim shell.sh
```

### 输入以下内容 :wq保存

```
#/bin/sh
k=$1
# -r 表示远程服务器发起调用，想使用本机调用时去掉-r即可
./bin/jmeter -n -t ./demo.jmx -r -l ./report/$k.csv -e -o ./report/$k
```

### 运行

```
$ ./shell.sh t1
```

### 等待执行完成，在report目录下就会生成一个名字为t1的文件夹，下载下来打开index.html就可以看到报告内容了
