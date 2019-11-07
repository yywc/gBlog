## 解压缩

解压到 /usr

```shell
tar -zxvf apache-tomcat-8.5.23.tar.gz -C /usr
```

## 配置环境变量

```shell
vim /etc/profile
```

在最下方添加

```shell
export CATALINA_HOME=/usr/apache-tomcat-8.5.23
```

## 配置 UTF-8 字符集

```shell
vim /usr/apache-tomcat-8.5.23/conf/server.xml
```

找到配置 8080 默认端口的位置，在 xml 节点末尾增加 URIEncoding="UTF-8"

## 启动 apache

``` shell
cd /usr/apache-tomcat-8.5.23/bin
./startup.sh
```
