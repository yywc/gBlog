## 前言

SSL 证书是数字证书的一种，遵守 SSL 协议，具有服务器身份验证和数据传输加密的功能。这里主要采用了免费的 let's enscript 来在服务器上进行配置。

## 1. 使用 acme.sh 来安装并生成证书

### 1.1  安装 acme.sh

acme.sh 会安装到 ~/.acme.sh 目录下

```shell
curl https://get.acme.sh | sh
// 如果出错
yum install -y socat
```

### 1.2  生成证书

注意**此处会占用 80 端口**，如果有其他进程占用了 80 端口要关闭，例如 nginx。
这里使用了 standalone 模式，此外还有 nginx apache 模式等，详情请看 https://github.com/Neilpang/acme.sh

```shell
~/.acme.sh/acme.sh --issue --standalone -d example.com -d www.example.com -d test.example.com (-k ec-256 可以选，-k 表示密钥长度)
```

### 1.3 更新证书

let's enscript 的有效期只有 90 天，到期需要更新证书。acme.sh 脚本 60 天会自动更新一次，当然也可以手动更新
手动更新

```shell
~/.acme.sh/acme.sh --renew -d example.com -d www.example.com -d test.example.com --force --ecc
```

### 1.4 安装证书和秘钥

多个域名只需要配置第一个就可以了，此处安装到 /etc/nginx/ssl/ 路径下(最好不好修改，手动更新的时候比较麻烦)

```shell
~/.acme.sh/acme.sh --install-cert --ecc -d example.com \
--key-file      /etc/nginx/ssl/example.com.key  \
--fullchain-file /etc/nginx/ssl/example.com.crt
```

## 2.  nginx 配置

```nginx
server {
  listen  443 ssl;
  ssl on;
  ssl_certificate      /etc/nginx/ssl/wangshukeji.crt;
  ssl_certificate_key  /etc/nginx/ssl/wangshukeji.key;
  ssl_protocols        TLSv1 TLSv1.1 TLSv1.2;
  ssl_ciphers          HIGH:!aNULL:!MD5;
  server_name          example.com;
}
# 配置 302 转发
server {
  listen  80;
  server_name  example.com;
  return 302 https://example.com$request_uri;
}
```

## 3.  通配符域名

```shell
// 阿里云控制台 accsess key 打开得到下面的值
export Ali_Key="key"
export Ali_Secret="secret"
acme.sh --issue --dns dns_ali -d example.com -d '*.example.com'

// Namesilo
export Namesilo_Key="xxxxxxxxxxxxxxxxxxxxxxxx"
acme.sh --issue --dns dns_namesilo --dnssleep 900 -d example.com -d '*.example.com'

acme.sh --renew -d example.com -d '*.example.com'
```

中间会等待120s，生成的证书在 /root/.acme.sh/example.com/ 目录下，然后配置 fullchain.cer 和 example.com.key 即可。
