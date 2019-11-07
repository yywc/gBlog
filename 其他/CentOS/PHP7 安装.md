# 下载
下载源码到 /opt 里面，接着 tar -zxvf 解压，首先安装一些准备装的扩展要用到的软件模块
```shell
yum -y install libjpeg libjpeg-devel libpng libpng-devel freetype freetype-devel libxml2 libxml2-devel zlib zlib-devel curl curl-devel openssl openssl-devel
```

# 安装
cd 到 /opt 下解压的 php 文件夹中，config
```shell
./configure --prefix=/usr/local/php-7.0.5 --enable-fpm --with-fpm-user=nginx --with-fpm-group=nginx --with-mysqli --with-pdo-mysql --with-zlib --with-curl --with-gd --with-jpeg-dir --with-png-dir --with-freetype-dir --with-openssl --enable-mbstring --enable-ftp --enable-zip
```
出现了 
> Thank you for using PHP.

就代表成功了
接着输入
```shell
make
make install
```
如果期间遇到内存不足的情况，采用以下添加 swap 分区的方式。
```shell
dd if=/dev/zero of=/var/swap bs=1024 count=1024000  // 增加 1G 的 swap 文件块
mkswap /swap  // 设置交换分区
swapon /swap  // 激活启用交换分区
swapon -s // 查看信息是否准确
echo "/var/swapfile swap swap defaults 0 0" >> /etc/fstab // 添加到fstab文件中让系统引导时自动启动
// 收回分区
swapoff/swapfile
rm -fr /swapfi
```
# 配置 PHP
将 php.ini 复制到响应位置
```shell
 cp php.ini-development /usr/local/php-7.x.x/lib/php.ini
```
修改 php.ini 配置
```shell
 vim /usr/local/php-7.x.x/lib/php.ini
// 查找 mysqli.default_socket，修改成：mysqli.default_socket = /var/lib/mysql/mysql.sock
// 查找 date.timezone 修改成 date.timezone = PRC，记得去掉注释
```
已经装好了，验证一下 
```shell
 /usr/local/php-7.x.x/bin/php -v 可以看到相关信息
```
# 方便配置（可选）
将 PHP 软链到某个不带版本号的文件夹
```shell
ln -s /usr/local/php-7.x.x /usr/local/php
ln -s /usr/local/php/bin/php /usr/sbin/php
ln -s /usr/local/php/bin/pecl /usr/sbin/pecl
ln -s /usr/local/php/bin/pear /usr/sbin/pear
ln -s /usr/local/php/bin/phpize /usr/sbin/phpize
// 省去路径，直接使用 php
 php -v
```
# 配置 php-fpm
```shell
cp /usr/local/php-7.x.x/etc/php-fpm.conf.default /usr/local/php-7.x.x/etc/php-fpm.conf
cp /usr/local/php-7.x.x/etc/php-fpm.d/www.conf.default /usr/local/php-7.x.x/etc/php-fpm.d/www.conf
 vim /usr/local/php-7.x.x/etc/php-fpm.d/www.conf
// 注意 user 和 group 是不是指向的 nginx 的用户名（默认是 nginx），注意监听端口是否为 listen = 127.0.0.1:9000
// 接着 cp，取名是方便以后升级
cp sapi/fpm/php-fpm.service /usr/lib/systemd/system/php-fpm-7xx.service
// 软链
ln -s /usr/lib/systemd/system/php-fpm-7xx.service /usr/lib/systemd/system/php-fpm.service
// 修改 php-fpm.service，把相对路径修改成绝对路径
vim /usr/lib/systemd/system/php-fpm.service
原：
PIDFile=${prefix}/var/run/php-fpm.pid
ExecStart=${exec_prefix}/sbin/php-fpm --nodaemonize --fpm-config ${prefix}/etc/php-fpm.conf
修改成
PIDFile=/usr/local/php-7.x.x/var/run/php-fpm.pid
ExecStart=/usr/local/php-7.x.x/sbin/php-fpm --nodaemonize --fpm-config /usr/local/php-7.x.x/etc/php-fpm.conf
// 重新载入
 systemctl daemon-reload
// 开机启动
systemctl enable php-fpm
// 启动
systemctl start php-fpm
ps： 如果出现了 php7 [pool www] cannot get uid for user 'nginx' 这样的错误，我用 yum 重装了一下 nginx 就解决了。估计是源码安装 nginx 哪里设置不对。 
```
# nginx 里配置
```nginx
server {
    listen       80;
    server_name  server_name;  // 访问地址
    root         /www;  // web 资源存放位置
    location / {
        index  index.php index.html index.htm;
    }
    location ~ \.php$ {
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        include        fastcgi_params;
    }
}
```
systemctl reload nginx 生效

