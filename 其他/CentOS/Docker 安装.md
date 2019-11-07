## 清除旧版本

```shell
yum remove docker \
           docker-client \
           docker-client-latest \
           docker-common \
           docker-latest \
           docker-latest-logrotate \
           docker-logrotate \
           docker-selinux \
           docker-engine-selinux \
           docker-engine
```

## 安装必要系统工具

```shell
yum install -y yum-utils device-mapper-persistent-data lvm2
```

## 添加软件源信息

```shell
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

## 更新 yum 缓存

```shell
yum makecache fast
```

## 安装 Dcoker-CE

```shell
yum -y install docker-ce
```

## 启动 Docker 服务

```shell
systemctl start docker
```

## 删除 Dcoker-CE

```shell
yum remove docker-ce
rm -rf /var/lib/docker
```
