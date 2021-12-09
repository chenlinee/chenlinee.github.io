---
title:  "Transmission安装与设置"
search: true
categories:
  - NAS
tags:
  - Transmission
---

Transmission是一款开源的BitTorrent客户端，它的核心下载程序与用户控界面程序分离，支持夸平台使用。

本文主要介绍在Debian上安装配置Transmission的核心下载程序和用户界面。

## 安装

Debian的官方源有`transmission`, `transmission-cli`, `transmission-common`, `transmission-daemon`, `transmission-gtk`, `transmission-qt`, `transmission-remote-gtk`等几个transmission相关安装包。`transmission-common`是核心的下载程序，但是不具备直接与用户交互的功能，需要配合transmission前端使用；`transmission-common`主要包含transmission前后端共享的必要组件；`transmission-cli`是一个命令行前端，可以通过命令行完成添加删除种子的操作，还有基于http的带有一个WebUI图形界面；`transmission-qt`, `transmission-remote-gtk`, `transmission-remote-gtk`是给桌面图形用户准备的。

如果是桌面用户，安装`transmission`包就会依赖安装核心下载程序后端和桌面图形用户界面前端。如果不需要图形界面功能，安装`transmission-daemon`包即可，apt会自动额外安装`transmission-cli`, `transmission-common`。我安装了不带桌面图形界面的`transmission`，用于支持NAS的种子下载。

## 配置

`transmission`的默认的下载目录、配置文件目录位于`/var/lib/transmission-daemon/`。

```bash
$ tree /var/lib/transmission-daemon/ -a -L 3
/var/lib/transmission-daemon/
├── .config
│   └── transmission-daemon
│       ├── blocklists
│       ├── dht.dat
│       ├── resume
│       ├── settings.json -> /etc/transmission-daemon/settings.json
│       ├── stats.json
│       └── torrents
├── downloads
└── info -> .config/transmission-daemon
```

配置文件就是其中的settings.json，实际上位于`/etc/transmission-daemon/settings.json`。transmission安装完成后，就已经处于运行状态并且设置成开机自启动。若要修改配置文件，应该先关掉transmission，再进行修改，否则修改的配置文件无法成功保存。

### 设置transmission下载目录

transmission程序安装时，新建了`debian-transmission : debian-transmission`用户和用户组，进程默认在这个用户下运行。为了确保下载的文件有权限保存在目录中，首先需要给`debian-transmissio`授权自定义下载目录的权限（默认下载目录的owner就是`debian-transmission:debian-transmission`），或者直接给下载目录设置0777权限。

```bash
# 新建下载完成和下载中目录
mkdir /dir/downloads /dir/Downloads
sudo chown -R debian-transmission:debian-transmission /dir/downloads /dir/Downloads
# 把自己的用户加到debian-transmission组，方便操作文件
sudo usermod -a -G debian-transmission example_user
```

### 设置下载目录、缓存大小等

trasnmission的[官方wiki](https://github.com/transmission/transmission/wiki/Editing-Configuration-Files)中介绍了每个配置项的意义。使用[PT下载](https://zh.wikipedia.org/wiki/PT%E4%B8%8B%E8%BC%89)时，建议关掉[PEX和DHT](https://zh.wikipedia.org/wiki/%E8%8A%82%E7%82%B9%E4%BA%A4%E6%8D%A2)。打开`/etc/transmission-daemon/settings.json`文件，修改下面的配置项：
```bash
# 下载完成的文件夹
"download-dir": "/dir/downloads",
# 下载未完成的文件夹
"incomplete-dir": "/dir/Downloads",
"incomplete-dir-enabled": true,
# 增大缓存大小，减小硬盘读写次数；不要设置太大，
# 避免意外关机导致过多缓存丢失
"cache-size-mb": 512,
# 监听端口
"peer-port": 51413,
# 下载文件夹的权限掩码。文件的权限是文件夹权限去掉可执行
# - 这里数值是10进制掩码，对应的是8进制掩码
# - 如，18 => 022, 对应文件夹权限是0755，文件权限是0644
#       2  => 02, 对应文件夹权限是0775，文件权限是0664
"umask": 2,
# PT下载
"dht-enabled": false,
"pex-enabled": false,
```

### 设置远程访问

transmission支持客户端远程（网络）控制下载进程，即RPC功能。打开`/etc/transmission-daemon/settings.json`文件，可以自定义RPC选项：
```bash
# 是否开启rpc服务（远程连接）
"rpc-enabled": true,
# 监听地址和端口
"rpc-bind-address": "0.0.0.0",
"rpc-port": 9091,
# 用户名-密码，daemon运行后会哈希密码
"rpc-username": "username",
"rpc-password": "passwd",
# 允许访问RPC的ip地址
"rpc-whitelist": "127.0.0.1, 192.168.*.*",
"rpc-whitelist-enabled": true,
```

#### 设置RPC的反向代理

进一步配置Apache2反向代理RPC，实现https加密访问transmission的RPC功能。

首先，开启Apache2的代理模块和SSL模块，并开启http强制跳转https。
```bash
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod ssl
sudo a2enmod rewrite
```

然后，在Apache2的site-available目录`/etc/apache2/sites-available/`中添加网站配置文件transmission.example.com.conf。
```config
<IfModule mod_proxy.c>
  <IfModule mod_ssl.c>
    <VirtualHost *:80>
      ServerName transmission.example.com
      Redirect permanent / https://transmission.example.com/
    </VirtualHost>

    <VirtualHost *:443>
      ServerName transmission.example.com

      SSLEngine on
      SSLCertificateFile /etc/ssl/transmission.example.com/cert.pem
      SSLCertificateKeyFile /etc/ssl/transmission.example.com/key.pem
      SSLCertificateChainFile /etc/ssl/transmission.example.com/fullchain.pem

      ProxyVia Full
      ProxyPass / http://localhost:9091/
      ProxyPassReverse / http://localhost:9091/
    </VirtualHost>
  </IfModule>
</IfModule>
```

启用反向代理配置文件，并重启Apache2，就可以通过 https://transmission.example.com/ 访问transmission的RPC了。

```bash
sudo a2ensite transmission.example.com
sudo systemctl restart apache2
```