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

## 申请证书

acme.sh 3.0版本之后，默认申请的证书时zerossl提供的。通过zerossl申请证书，需要先用acme.sh脚本注册账号。如果选择Let's Encrypt申请证书，就不用注册了。

```bash
acme.sh --register-account="email@mail.com"
```

### 通过DNS解析申请泛域名证书

泛域名证书需要提供DNS解析证明你是域名的拥有者，不需要通过设置本机的http服务器验证域名ip对应关系，对内网机器非常友好。[acme.sh支持100+ DNS记录提供商的API接口](https://github.com/acmesh-official/acme.sh/wiki/dnsapi)，设置好API即可通过acme.sh自动申请证书。

我的域名解析托管商是阿里云，在阿里云控制台申请到API后，设置临时环境变量：

```bash
export Ali_Key="sdfsdfsdfljlbjkljlkjsdfoixxx"
export Ali_Secret="jlsdflanljkljlfdsaklkjfxxxx"
```

然后使用对应DNS验证申请证书，acme.sh会自动缓存API信息。在实际申请时，我通过zerossl申请证书出现了`504 Gateway Timeout`错误，可能是网络问题，手动指定服务器换成原来的Let's Encrypt顺利申请。

acme.sh申请证书的长度默认是rsa-2048，这里改成ecc-256，可以减少秘钥计算，提高安全性。

```bash
# a.example.com, b.example.com共享证书
acme.sh --issue --server letsencrypt --dns dns_ali -k ec-256 -d a.example.com -d b.example.com
# 为c.example.com生成单独的证书
acme.sh --issue --server letsencrypt --dns dns_ali -k ec-256 -d c.example.com
```

执行完成之后，即可申请到证书。通过`acme.sh --list`命令就能看到申请的证书了。

## acme.sh证书安装与更新

acme.sh生成的证书默认放在~/.acme.sh文件夹，acme.sh后续的更新可能会更改内部文件结构。因此不推荐直接使用这个文件夹的证书，而应该安装证书到其他目录。

```bash
acme.sh --install-cert --ecc -d example.com \
--cert-file /dir/example.com/cert.pem \
--key-file /dir/example.com/key.pem \
--fullchain-file /dir/example.com/fullchain.pem \
--reloadcmd "bash /dir/example.com/reload.sh"
```

`--reloadcmd`参数可以提供一个执行更新命令的接口，在这里执行脚本，实现更新证书时完成一些自动化任务，比如让nginx重新加载证书。


