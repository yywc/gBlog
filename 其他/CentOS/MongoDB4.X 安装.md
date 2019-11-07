## 下载

通过 [mongodb官网](https://www.mongodb.com/download-center/community) 进行下载，这里下载 RHEL 7.0 Linux 64-bit x64 版本，复制下载链接到 /opt 目录下，然后运行

```shell
> wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel70-4.0.3.tgz
```

## 解压安装

```shell

> tar -zxvf mongodb-linux-x86_64-rhel70-4.0.3.tgz

> mv mongodb-linux-x86_64-rhel70-4.0.3.tgz /usr/local/mongodb

// 下面步骤是创建日志目录和数据目录

> cd /usr/local/mongodb

> mkdir -p data/db

> mkdir log

> touch mongodb.log

// 创建启动配置文件

> vim bin/mongodb.conf

logpath=/usr/local/mongodb/log/mongodb.log

dbpath=/usr/local/mongodb/data/db

logappend=true

fork=true

port=27017

bind_ip=0.0.0.0

#auth=true // 暂时先注释，配置完 mongodb 后再开启安全验证
```

## 启动、重启、停止

```shell

// 配置环境变量
> vim /etc/profile
export MONGODB_HOME=/usr/local/mongodb
export PATH=$PATH:$MONGODB_HOME/bin
> source /etc/profile

// 配置系统服务
> vim /usr/lib/systemd/system/mongodb.service
[Unit]
Description=mongodb
After=network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
ExecStart=/usr/local/mongodb/bin/mongod -f /usr/local/mongodb/bin/mongodb.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/usr/local/mongodb/bin/mongod --shutdown -f /usr/local/mongodb/bin/mongodb.conf
PrivateTmp=true

[Install]
WantedBy=multi-user.target

// 可以使用系统命令来操作
> systemctl start mongodb
> systemctl restart mongodb
> systemctl stop mongodb
```

## 初始化数据库

由于我们的配置文件注释掉了 auth，所以进入时不需要验证就有所有的权限，接下来就是配置用户和数据库了。

```shell
> mongo

> use admin

> db.createUser({user:'admin',pwd:'admin',roles:[{role:'root',db:'admin'}]}) // 创建 admin 超级管理员

> use test

> db.createUser({user:'test',pwd:'test',roles:[{role:'dbOwner',db:'test'}]}) // 给 test 数据库创建角色

> exit  // 角色相关的信息可以通过 https://docs.mongodb.com/manual/core/security-built-in-roles/index.html 来查询

> vim /usr/local/mongodb/bin/mongodb.conf // 去除 auth 前的注释

> mongo

> use test

> db.auth('test', 'test') // 1

> db.Test.insert({name: 'hello world!'}) // WriteResult({ "nInserted" : 1 })

> db.Test.find() // { "_id" : ObjectId("5bd958e83bb558935e55cb9e"), "name" : "hello world" }

```

至此就完成了基本的配置了。
