# 安装

(注意，以下都在 root 账号进行的操作，if not，sudo)

执行安装命令

```shell
yum -y install vsftpd
```

# 在根目录创建 ftpfile 用来保存上传的文件

```shell
cd /
mkdir ftpfile
```

# 添加用户

仅有传输权限，没有登录权限

```shell
useradd ftpuser -d /ftpfile -s /sbin/nologin
```

# 赋权

```shell
chown -R ftpuser.ftpuser /ftpfile
```

查看 ftpfile 所在用户

```shell
ll | grep ftpfile
```

# 设置密码

```shell
passwd ftpuser
```

出现警告忽略即可

# 添加用户

找到 vsftpd.conf,路径默认是 /etc/vsftpd/vsftpd.conf

+ lisrten=YES 那么就要关闭 listen_ipv6

+ 找到 anonymous_enable=YES 改为 NO // 拒绝匿名登录

+ 打开 ftpd_banner=Welcome to happymmal FTP service.

+ 新增 local_root=/ftpfile // 本地目录指向创建文件夹

+ 新增 use_localtime=YES // 服务器使用本地时间

+ 打开 chroot_list_enable=YES

+ 打开 chroot_list_file=/etc/vsftpd/chroot_list 

+ cd 到 /etc/vsftpd，新建 vim chroot_list，添加 ftpuser，保存

+ pasv_min_port=61001 pasv_max_port=62000

+ listen=NO listen_ipv6=YES

+ 打开 chroot_local_user= NO 新增 allow_writeable_chroot=YES


# 修改安全文件

+ 修改 /etc/selinux/config

```shell
vim /etc/selinux/config
SELINUX=disabled
```

+ 打开 ftp 防火墙服务

```shell
firewall-cmd --permanent --zone=public --add-service=ftp
firewall-cmd --reload
```

# 重启

```shell
systemctl restart vsftpd.service
```

# 相关命令

```shell
systemctl restart vsftpd.service  # 重启服务
systemctl start vsftpd.service    # 启动服务
systemctl status vsftpd.service   # 服务状态查看
systemctl enable vsftpd.service   # 设置自启
```