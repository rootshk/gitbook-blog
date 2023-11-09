# \[HomeKit] ESP-01S使用DHT11制作温湿度传感器

`Date: 2023-10-11 10:23 北京时间`&#x20;

`Device: MacBook Pro 14 2021`&#x20;

`OS: macOS 14.0`&#x20;

`Soft: Python3`



**下载安装工具esptool**

```
mkdir esp01
cd esp01
git clone https://github.com/espressif/esptool.git
cd esptool
```

**下载固件fullhaaboot**

```
wget https://github.com/RavenSystem/haa/releases/latest/download/fullhaaboot.bin
```

**插入刷写器 查询USB端口号**

```
ls /dev/cu.*
```

**清除ESP-01S内容**

```
python3 esptool.py -p /dev/cu.usbserial-<PROT> erase_flash
```

**刷写固件**

```
python3 esptool.py -p /dev/cu.usbserial-<PROT> --baud 115200 write_flash -fs 1MB -fm dout -ff 40m 0x0 fullhaaboot.bin
```

**连接配置**

```
1. 连接Wifi HAA-****
2. Search Wifi后选择对应Wifi
3. 设置Wifi密码
4. 配置Json
```

```
{ "a":[ { "t":24, "g":0, "n":2 } ] }
```

```
5. 点击Save
```

**打开iPhone Home App**

```
1. 添加配件
2. 扫码或者输入编号
```

![2023-10-11T02:30:36.png](https://roothk.top/usr/uploads/2023/10/1152151854.png)
