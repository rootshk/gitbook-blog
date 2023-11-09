# \[记录]CentOS离线安装记录

> 最近在出差啊, 就有遇到说甲方机器是内网的, linux这些没法用yum的包管理器在线安装一些软件

#### 解决方案

**一. 下载源码编译**

```
这个方法当然这样是最好的啦, 但是我懒, 所以看方法二
```

**二. 用YumDownloader获取离线安装包**

**1) 安装YumDownloader**

找一台centos系统的机器(对应你要离线安装的系统), 可以是服务器,可以是虚拟机,实体机啥的. 而且是要已联网的,并执行命令

```
# 安装
yum install yum-utils -y
# 建立一个空文件夹
mkdir install_file & cd install_file
```

**2) 使用YumDownloader获取离线安装包**

运行下面命令获取, 可以自己替换后面的软件名称, 这里以git做演示

```
yumDownloader git
```

**3) 打包离线安装包**

这是时候你ll一下你的目录就会看到有许多的.rpm文件, 我们可以把它们都打包起来,塞入需要安装的服务器执行安装就可以了

```
# 打包
zip –q –r install_file.zip *
```

**4) 安装**

这时候你把刚刚压缩的文件放到你需要离线安装的服务器里面(并cd到对应目录),解压

```
unzip install_file.zip
```

然后执行安装

```
rpm -ivh *.rpm --force --nodeps
```
