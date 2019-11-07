## 安装 gcc pcre zlib openssl

```shell
gcc -v (如果没有安装，如需支持 ssl，才安装 openssl)
yum -y install gcc pcre-devel zlib zlib-devel openssl openssl-devel
```

## yum 安装

### 更新 yum repo

```shell
rpm -Uvh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
如果 404， 修改 hosts 指向 nginx.org 的 ip 地址
```

### 安装并设置

```shell
yum install nginx
nginx -v

// 开机启动
systemctl enable nginx

// 开启
systemctl start nginx

// 检查
systemctl status nginx
```

## 源码安装

### 下载

```shell
wget http://nginx.org/download/nginx-1.12.2.tar.gz
```

### 解压

```shell
tar -zxvf nginx-1.12.2.tar.gz
```

进入 nginx 目录执行

```shell
./configure --prefix=/usr/nginx
make
make install
```

### 常用命令

```shell
/usr/nginx/sbin/nginx -t    # 测试配置文件

/usr/nginx/sbin/nginx    # 启动

/usr/nginx/sbin/nginx -s stop / quit    # 快速停止 / 完整有序的停止

/usr/nginx/sbin/nginx -s reload    # 重启

ps -ef | grep nginx    # 查看进程

xxx

kill -HUP xxx    # 平滑重启

```

## 防火墙

```shell
firewall-cmd --zone=public --add-port=80/tcp --permanent
firewall-cmd --reload
```

## 虚拟域名配置

```shell
vim /etc/nginx/nginx.conf || vim /usr/nginx/conf/nginx.conf

/deny (找到这一行 } 下添加)

include vhost/*.conf;
```

在 /usr/nginx/conf 下新建 vhost 文件夹，在 vhost 中创建域名转发配置文件

## 卸载

查询 nginx目录

+ find    /  -name '*nginx*' //  查询 /目录下 文件名包含nginx的文件路径

+ 使用rm 命令卸载 nginx

+ rpm -q -a | grep nginx    查询安装软件 package name

+ 确定了要卸载的软件的名称，就可以开始实际卸载该软件了。键入命令： rpm -e  [package name]
