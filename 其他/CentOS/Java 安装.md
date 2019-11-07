## 安装前

清理系统自带的 jdk (openjdk)

```shell
rpm -qa | grep jdk
(查询结果为 xxx)
yum remove xxx
```

## 赋权

```shell
chmod 777 jdk-8u152-linux-x64.rpm
```

## 安装

```shell
rpm -ivh jdk-8u152-linux-x64.rpm
```

## 安装路径

默认为 /usr/java

## 环境变量

```shell
vim /etc/profile
```

在最下方添加

```shell
export JAVA_HOME=/usr/java/jdk1.8.0_152

export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar

export PATH=$JAVA_HOME/bin:$PATH
```

## 确定生效

```shell
source /etc/profile
```
