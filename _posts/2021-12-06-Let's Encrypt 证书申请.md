---
title:  "Let's Encrypt 证书申请"
search: true
categories:
  - 网站搭建
tags:
  - Let's Encrypt
header:
#    teaser: /assets/images/2021-11-28-Linux内核简介/Linux_kernel_development.jpg
---

Let's Encrypt提供免费的SSL证书申请，记录申请泛域名证书的步骤。

## 使用acme.sh脚本申请证书

Let's Encrypt使用acme协议颁发证书，支持acme协议的客户端很多，其中acme.sh是一个用bash脚本写的客户端，安装使用方便。使用在线方式安装acme.sh脚本：

```bash
curl https://get.acme.sh | sh
source ~/.bashrc
```

脚本安装时，自动检测必要组件，没有的话安装一下即可。安装完成后，acme.sh在用户的$HOME目录生成.acme.sh文件夹，脚本的所有文件都会放在这个文件夹中，不会污染其他目录。同时，脚本在`~/.bashrc`添加`alias acme.sh=~/.acme.sh/acme.sh`，安装完成后执行`source ~/.bashrc`即可直接使用`acme.sh`命令。

由于Let's Encrypt免费证书3个月自动过期，acme.sh安装时，还会设置一个crontab定时工作，每天检查一次证书是否快要过期，完成自动更新。目前，acme.sh的证书更新时间是证书颁发日期60天后。

### 注册zerossl（可选）

acme.sh 3.0版本之后，默认申请的证书时zerossl提供的。通过zerossl申请证书，需要先用acme.sh脚本注册账号。如果选择Let's Encrypt申请证书，就不用注册了。

```bash
acme.sh --register-account="email@mail.com"
```

## 申请证书的方式

acme.sh 支持多种申请证书的方式，各有适合的场景。

```bash
# 假设需要申请证书的域名是 example.com
domain_name=example.com
```

### 通过DNS解析申请泛域名证书

泛域名证书需要提供DNS解析证明你是域名的拥有者，不需要通过设置本机的http服务器验证域名ip对应关系，对没有公网ip的机器友好。[acme.sh支持100+ DNS记录提供商的API接口](https://github.com/acmesh-official/acme.sh/wiki/dnsapi)，设置好API即可通过acme.sh自动申请证书。

我的域名解析托管商是CloudFlare，在CloudFlare申请到相关API后，设置临时环境变量：

教程：https://github.com/acmesh-official/acme.sh/wiki/dnsapi#dns_cf

```bash
export CF_Token=${cf_token}
export CF_Zone_ID=${cf_zone_id}
export CF_Account_ID=${cf_account_id}
```

然后使用对应DNS验证申请证书，acme.sh会自动缓存API信息。通过zerossl申请证书遇到`504 Gateway Timeout`的网络问题，切换服务器为Let's Encrypt顺利申请。

acme.sh申请证书的长度默认是rsa-2048，这里改成ecc-256，可以减少秘钥计算，提高安全性。

```bash
# a.example.com, b.example.com共享证书
acme.sh --issue --server letsencrypt --dns dns_cf -k ec-256 -d a.example.com -d b.example.com
# 为c.example.com生成单独的证书
domain_name=example.com
acme.sh --issue --server letsencrypt --dns dns_cf -k ec-256 -d ${domain_name}
```

执行完成之后，即可申请到证书。通过`acme.sh --list`命令就能看到申请的证书了。

### 通过 http 方式申请证书

acme.sh 默认 http 方式需要依赖 web 服务器，在指定目录放置文件完成验证。
acme.sh 也可以直接接模拟 web 服务器，监听 80 端口完成验证，实现证书签发。模拟 web 服务器
的方式更方便。为了避免和 web 服务器端口冲突，可以使用 nginx 的 `proxy` 模式。

```bash
port=21742
sudo sh -c 'cat << EOF > /etc/nginx/sites-available/let_encrypt
server {
    listen       80;
    server_name  '"${domain_name}"';

    location / {
        proxy_pass http://127.0.0.1:'"${port}"';
    }
}
EOF'
sudo ln -s /etc/nginx/sites-available/let_encrypt /etc/nginx/sites-enabled/let_encrypt
sudo systemctl reload nginx
```

设置后， acme.sh 改用端口申请证书。

```bash
acme.sh --issue -d ${domain_name} -k ec-256 --standalone --httpport ${port}
```

## acme.sh证书安装与更新

acme.sh生成的证书默认放在~/.acme.sh文件夹，acme.sh后续的更新可能会更改内部文件结构。因此不推荐直接使用这个文件夹的证书，而应该安装证书到其他目录。

```bash
install_path=/etc/acme_ssl/certs
mkdir -p ${install_path}
chmod 700 ${install_path}
acme.sh --install-cert --ecc -d ${domain_name} \
--key-file ${install_path}/${domain_name}.key \
--fullchain-file ${install_path}/${domain_name}.crt \
--ca-file ${install_path}/${domain_name}.ca.crt \
--reloadcmd "systemctl reload nginx"
```

`--reloadcmd`参数可以提供一个执行更新命令的接口，在这里执行脚本，实现更新证书时完成一些自动化任务，比如让nginx重新加载证书。


