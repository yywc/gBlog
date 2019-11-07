## 下载

`http://repo.mysql.com/` 上找到合适的版本，此处用 `http://repo.mysql.com/mysql57-community-release-el7-11.noarch.rpm`

```shell
wget http://repo.mysql.com/mysql57-community-release-el7-11.noarch.rpm
sudo rpm -ivh mysql57-community-release-el7-11.noarch.rpm
```

## 安装

采用 yum 安装，简单便捷

```shell
# 更新yum软件包
yum check-update

# 更新系统
yum update

#安装mysql
yum install mysql mysql-server
```

## 编辑配置文件

```shell
vim /etc/my.cnf

# 在 [mysqld] 节点下添加
character-set-server=utf8
```

## 防火墙设置

```shell
# 开放 3306 端口
firewall-cmd --zone=public --add-port=3306/tcp --permanent
firewall-cmd --reload
```

## 启动登录

```shell
systemctl start mysqld

# 拿到临时密码登录
sudo grep 'temporary password' /var/log/mysqld.log

# 输入密码
mysql -u root -p

# 设置密码 (格式为大小写加特殊符号，这里先按预设的设置，到时候修改即可)
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4!';

# 设置账户密码成功，删除密码检验规则，设置自由密码
mysql> uninstall plugin validate_password;
mysql> SET PASSWORD FOR 'root'@'localhost' = PASSWORD('newpass');

# 如果需要装回密码校验规则
mysql> INSTALL PLUGIN validate_password SONAME 'validate_password.so';
```

## 相关操作

```shell
# 本地登录
mysql> CREATE USER "test"@"localhost" IDENTIFIED BY "1234";

# 远程登录
mysql> CREATE USER 'test'@'%' IDENTIFIED BY '1234';
mysql> quit

# 测试是否创建成功
mysql> mysql -u test -p

# 新建用户
CREATE USER 'username'@'host' IDENTIFIED BY 'password';

# 为用户创建偶一个数据库
mysql> create database 数据库名称 default charset utf8 collate utf8_general_ci;

# 用户授权—全部授权 (2种)
mysql> grant all privileges on 数据库名称.* to "用户名"@"主机" identified by "密码";
mysql> GRANT ALL PRIVILEGES ON `%`.* TO 'root'@'localhost' IDENTIFIED BY 'password' WITH GRANT OPTION;

# 用户授权—部分授权
mysql> grant select,update on testDB.* to"test"@"localhost" identified by "1234";

# 刷新系统权限表
mysql> flush privileges;

# 删除用户
mysql> DELETE FROM mysql.user Where User="用户名" and Host="主机";
mysql> flush privileges;
mysql> drop database 数据库名称;

# 删除账户及权限
drop user 用户名@’%’;
drop user 用户名@ localhost;

# 查看目前 mysql 的用户
mysql> select user,host from mysql.user;

# 查看是否有匿名用户
mysql> select user,host from mysql.user;

# 删除匿名用户
mysql> delete from mysql.user where user='';

# 刷新权限
mysql> flush privileges;

# 展示所有数据库
mysql> show databases;

# 使用某个数据库
mysql> use 数据库名

# 创建表 (建立一个员工的生日表，表的内容包含员工姓名、性别、出生日期、出生城市)，更多的操作可以在数据库图形化界面中进行操作，如 DataGrip
mysql> create table mytable (name varchar(20), sex char(1),birth date, birthaddr varchar(20));
```
