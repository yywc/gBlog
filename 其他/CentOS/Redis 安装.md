# 下载

```shell
yum install redis
```

# 启动

```shell
systemctl start redis
```

# 关闭

```shell
systemctl stop redis
```

# 开机启动

```shell
systemctl enable redis
```

# 开启外网访问

修改配置文件
```shell
vim /etc/redis.conf
// 修改成以下
# bind 127.0.0.1 
protected-mode no
```