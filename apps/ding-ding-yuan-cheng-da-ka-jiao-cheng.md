# 钉钉远程打卡教程

> 项目源码: https://github.com/rootshk/dding

#### 一 准备

**硬件准备: Android设备N台, 支持Shell服务器一台, 数据线N条**

**软件准备:**

> Android设备安装”钉钉”,并登陆 服务器安装ADB, JDK8+

**网络准备: 公网服务器一台(如果Shell服务器和公网服务器在同一lan,则需要代理, 如果Shell服务器是内网服务器, 则需要内网穿透)**

#### 二 连接

**将准备好的Android连接到Shell服务器, 开启USB调试,并信任Shell服务器**

**Shell运行dding程序:**

```
nohup java -jar dingding-0.0.1-SNAPSHOT.jar &
```

**访问IP:9603即可开始(用户密码均为:root)**
