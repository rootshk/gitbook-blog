# \[LOG] Maven打包记录

> 多子项目打包

```
# CORE_NUMBER获取系统逻辑处理器数量
CORE_NUMBER=$(cat /proc/cpuinfo | grep "processor" | wc -l)
# 子项目路径
fullPath=service/base
# -pl 按项目名编译 -am 编译该项目依赖的项目 -DskipTests=true 跳过测试 -T 多线程设置
mvn clean package -pl $fullPath -am -DskipTests=true -T $CORE_NUMBER
```
