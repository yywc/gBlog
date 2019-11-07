# 安装前

安装好 java，配置好环境变量

# 解压

解压到 /usr
```shell
tar -zxvf apache-maven-3.5.2-bin.tar.gz -C /usr
```

# 配置环境变量

```shell
vim /etc/profile
```

在最下方添加

```shell
export MAVEN_HOME=/usr/apache-maven-3.5.2
export PATH=$JAVA_HOME/bin:$MAVEN_HOME/bin:$PATH
```

# 确定验证

```shell
source /etc/profile
mvn -version
```