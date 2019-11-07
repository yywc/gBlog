## 下载

前往 https://github.com/git/git/releases 找到对应的版本,，右键 tar.gz 获取下载链接

```shell
wget https://github.com/git/git/archive/v2.15.0.tar.gz
```

## 安装依赖

```shell
yum -y install gcc pcre-devel zlib zlib-devel openssl openssl-devel expat-devel gettext-devel curl-devel perl-ExtUtils-CBuilder perl-ExtUtils-MakeMaker
```

## 解压安装

```shell
tar -zxvf v2.15.0.tar.gz
cd v2.15.0
make prefix=/usr all
make prefix=/usr install

# git version 2.15.0
git --version
```

## 基础配置

```shell
# 配置用户名
git config --global user.name "用户名"

# 配置邮箱
git config --global user.email "邮箱"

# 其他配置
(如果没有安装 kdiff3就不用)
git config --global merge.tool "kdiif3"
(让 git 不管 Windows/Unix 换行符转换)
git config --global core.autocrlf false

# 编码配置
gitconfig --global gui.encoding utf-8
git config --global core.quotepath off

# 查看配置
git config --list
```

## 生成秘钥

```shell
ssh-keygen -t rsa -C "邮箱"
ssh-add ~/.ssh/id_rsa

## 如果添加报错
eval `ssh-agent`

## 查看秘钥
cat ~/.ssh/id_rsa.pub
```
