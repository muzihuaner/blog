---
title: 使用 Docker 构建 AnyLink 企业级异地组网 及客户端配置全攻略
date: 2025-02-11 19:01:27
categories:
- 教程
tags:
- Docker
- Anylink
---
> AnyLink是一个企业级远程办公 异地组网 软件，可以支持多人同时在线使用。基于 openconnect 协议开发，并且借鉴了 ocserv 的开发思路，可以完全兼容 AnyConnect 客户端。

![online](https://gcore.jsdelivr.net/gh/muzihuaner/huancdn@main/img/online.jpg)

开源仓库：https://github.com/bjdgyc/anylink

**AnyLink** 最主要的功能莫过于`用户分组功能`.通过设置分组权限可以使不同用户或员工能访问的网段和路由也不同,加上支持 **SMTP** 邮件来分发密码,同时也支持 **TOTP 动态令牌** ,确实更适合小型企业的远程访问管理.但是`并不能`像 **Openconnect** 一样支持证书登录,**所以每次连接都需要手动输入密码**.

## 服务端安装

#### 1.安装Docker环境&解析域名

参考 https://getdocker.quickso.cn/

将你的域名解析到服务器

#### 2.拉取镜像

```
docker pull bjdgyc/anylink:latest
```

#### 3.生成密码和 jwt secret

首次运行以下命令来创建管理员密码.以下命令中含`rm`参数,仅用于执行创建密码的命令,会自行删除.将生成的`Passwd`和`secret`保存.

```
docker run -it --rm bjdgyc/anylink tool -p your_password
# 生成 Passwd
# Passwd:$2a$10$iKPApINPOwuyUhZC/b7P0eI8chEukkpyxPEbYxqdS3BPTtImqStGW

docker run -it --rm bjdgyc/anylink tool -s
# 生成 jwt secret
# Secret:r4-21-eDOrSAjtXL4k06PK0NFudNshMKh9euV09EQrsa9oEYlKaSgoa-CdY
```

![img](https://gcore.jsdelivr.net/gh/muzihuaner/huancdn@main/img/3933217940.jpg)

#### 4.获取配置目录

首先创建一个文件夹

```
mkdir AnyLink & cd AnyLink
```

复制以下命令,首次启动容器,拷贝配置文件到宿主机后删除容器.

```bash
docker run -itd --name anylink --privileged=true bjdgyc/anylink
```

使用 docker cp 将配置文件目录拷贝到当前目录

```bash
docker cp anylink:/app/conf .
```

删除容器

```
docker stop anylink
docker rm anylink
```

#### 5.正式启动容器

将 docker cp 拷贝出来的`conf`目录中的`server.toml`，进行修改

1. 修改第3步生成的密码和jwt secret，系统名称 

   ```
   issuer = "XXX-V*N系统"
   admin_user = "admin"
   admin_pass = "$2a$10$xxxxxxxxxxxxxxxxxxxxxx"
   jwt_secret = "SF6J8UnQAyxxxxxxxxxxxxxxxxx"
   
   ```

2. 虚拟网段

   ```
   #虚拟网络类型[tun macvtap]
   link_mode = "tun"
   #客户端分配的ip地址池
   #docker环境一般默认 eth0，其他情况根据实际网卡信息填写
   ipv4_master = "eth0"
   ipv4_cidr = "10.0.0.0/24"
   ipv4_gateway = "10.0.0.1"
   ipv4_start = "10.0.0.2"
   ipv4_end = "10.0.0.254"
   ```

   

参考以下命令正式启动容器（在`AnyLink`文件夹下）

```bash
docker run -d \
    --name anylink \
    --restart always \
    --privileged=true \
    -p 8443:443 \
    -p 8800:8800 \
    -v ./conf:/app/conf \
bjdgyc/anylink -c=/app/conf/server.toml
```

#### 6.管理后台

访问`内网IP:端口`,例如`https://IP:8800`输入`admin`密码登陆后台管理系统.

`https://IP:8443`即为首页，可以自定义

后台管理非常简单明了,只需要配置`用户组`,`用户`即可开始使用,如需邮件分发密码功能可以在`邮件设置`中配置.

![image-20250211191205518](https://gcore.jsdelivr.net/gh/muzihuaner/huancdn@main/img/image-20250211191205518.png)

`其他设置`中可以配置Banner信息、自定义首页（支持HTML代码）、邮件分发模版等，可以根据自己的需求修改

![image-20250211191032651](https://gcore.jsdelivr.net/gh/muzihuaner/huancdn@main/img/image-20250211191032651.png)

#### 7.用户组信息

一般保持默认即可，你也可以自定义

`添加用户组`只需要填写组名,其他保持默认,则表示该组用户所有流量都将被服务器代理,属于完整的`全局代理`.

![image-20250209113308300](https://gcore.jsdelivr.net/gh/muzihuaner/huancdn@main/img/image-20250209113308300.png)

`包含路由`如果添加内网网段,例如网段为`10.0.0.0/24`,则表示该组用户**可以访问内网设备**,而互联网请求依旧使用`客户端本地的网络访问`.

![image-20250211191743260](https://gcore.jsdelivr.net/gh/muzihuaner/huancdn@main/img/image-20250211191743260.png)

`DNS 服务器`可根据需求自行设置

#### 8.用户信息

`添加新用户`并且分配**用户组**,`PIN 码`为客户端的登录密码,如果配置了 **SMTP** 并勾选了`发送邮件`,则会创建随机密码以邮件的形式发送.也可以手动设置密码.`OTP`功能如果没有需求关闭即可.

![image-20250209113446838](https://gcore.jsdelivr.net/gh/muzihuaner/huancdn@main/img/image-20250209113446838.png)

自定义首页模板

```
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>XXX异地组网系统</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
            font-family: 'Segoe UI', Arial, sans-serif;
        }

        body {
            background-color: #f0f4f8;
            line-height: 1.6;
        }

        .header {
            background: linear-gradient(135deg, #1a73e8, #0d47a1);
            color: white;
            padding: 2rem 1rem;
            text-align: center;
            box-shadow: 0 2px 10px rgba(0,0,0,0.1);
        }

        .container {
            max-width: 1200px;
            margin: 2rem auto;
            padding: 0 1rem;
        }

        .card {
            background: white;
            border-radius: 10px;
            padding: 2rem;
            margin-bottom: 1.5rem;
            box-shadow: 0 2px 15px rgba(0,0,0,0.1);
        }

        .download-btn {
            display: inline-block;
            background: #1a73e8;
            color: white;
            padding: 12px 30px;
            border-radius: 25px;
            text-decoration: none;
            font-weight: bold;
            transition: transform 0.2s;
            margin: 1rem 0;
        }

        .download-btn:hover {
            transform: translateY(-2px);
            background: #1557b0;
        }

        h2 {
            color: #1a73e8;
            margin-bottom: 1rem;
            border-left: 4px solid #1a73e8;
            padding-left: 1rem;
        }

        ul {
            list-style: none;
            padding: 1rem;
        }

        li {
            padding: 0.5rem 0;
            display: flex;
            align-items: center;
        }

        li::before {
            content: "✔";
            color: #1a73e8;
            margin-right: 1rem;
        }

        .notice {
            background: #fff3cd;
            padding: 1rem;
            border-radius: 5px;
            margin: 1rem 0;
            border-left: 4px solid #ffc107;
        }

        footer {
            text-align: center;
            padding: 1.5rem;
            background: #1a273e;
            color: white;
            margin-top: 3rem;
        }

        @media (max-width: 768px) {
            .container {
                padding: 0 0.5rem;
            }
            
            .card {
                padding: 1.5rem;
            }
        }
    </style>
</head>
<body>
    <div class="header">
        <h1>XXX异地组网系统</h1>
        <p>安全连接 · 高效访问</p>
    </div>

    <div class="container">
        <div class="card">
            <h2>客户端下载</h2>
            <a href="https://ocserv.yydy.link:2023/" class="download-btn">立即下载异地组网客户端</a>
            <p>支持版本：Windows / macOS / Linux / Android / iOS</p>
        </div>

        <div class="card">
            <h2>支持客户端</h2>
            <ul>
                <li>Cisco AnyConnect 客户端</li>
                <li>OpenConnect 客户端</li>
                <li>兼容标准SSL V*N协议</li>
            </ul>
        </div>

        <div class="card">
            <h2>使用说明</h2>
            <div class="notice">
                <p>🔔 首次使用请先下载安装客户端，使用公司账号登录</p>
            </div>
            <ol style="padding-left: 2rem;">
                <li>下载并安装对应客户端软件</li>
                <li>输入服务器地址：https://V*N.abc.com:8443</li>
                <li>使用公司分配的账号密码登录</li>
                <li>连接成功后即可访问内部资源</li>
            </ol>
        </div>
    </div>

    <footer>
        <p>© 2026 XXX 版权所有</p>
        <p>技术支持：admin@xxx.com</p>
    </footer>
</body>
</html>

```



## 客户端安装

#### 下载客户端

[AnyConnect Secure Client](https://www.cisco.com/site/us/en/products/security/secure-client/index.html) ( Windows/macOS/Linux/Android/iOS)

[OpenConnect](https://gitlab.com/openconnect/openconnect) (Windows/macOS/Linux)

[三方 AnyLink Secure Client](https://github.com/tlslink/anylink-client) (Windows/macOS/Linux)

【推荐】三方客户端下载地址( Windows/macOS/Linux/Android/iOS) [国内地址](https://ocserv.yydy.link:2023/) [国外地址](https://cisco.yydy.link/#/)

#### 在Linux上安装（NAS或者无UI服务器）

###### 安装客户端（Redhat系列）

```
dnf install openconnect
```

###### 安装客户端（Debian系列）

```
apt install openconnect
```

##### 创建 systemd 服务文件

创建一个新的服务配置文件： `sudo nano /etc/systemd/system/openconnect-vpn.service`

将以下内容粘贴进去（已根据你提供的信息填充）：

```
[Unit]
Description=OpenConnect VPN Auto-start
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
# 使用 sh -c 来通过管道传递 PIN 码,将密码、用户名、用户组、服务器地址修改成你的
ExecStart=/bin/sh -c 'echo "password" | /usr/sbin/openconnect \
    --protocol=anyconnect \
    --user=tom \
    --authgroup=ops \
    --passwd-on-stdin \
    https://vpn.xxx.com:8443'

# 如果连接断开，5秒后自动重启服务
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> **注意**：请通过 `which openconnect` 确认你的 `openconnect` 二进制路径。如果是 `/usr/local/bin/openconnect`，请相应修改上面的路径。

------

##### 设置权限与启用服务

为了保护包含 PIN 码的配置文件，建议设置权限：

1. **设置权限**： `sudo chmod 600 /etc/systemd/system/openconnect-vpn.service`
2. **重新加载 systemd 配置**： `sudo systemctl daemon-reload`
3. **启动并设置开机自启**： `sudo systemctl enable --now openconnect-vpn`

------

##### 验证状态

你可以通过以下命令检查 是否连接成功：

- **查看服务状态**：`systemctl status openconnect-vpn`
- **查看实时日志**：`journalctl -u openconnect-vpn -f`
- **测试网络**：`ping` 一下内部的 IP 地址或检查 `ip addr` 是否出现了 `tun0` 网卡。



## 使用其他客户端

安装客户端后，输入服务器地址（有端口加端口）

点击连接，输入用户组、用户名、密码连接即可...
