---
title: DNS劫持解决方法
date: 2024-04-02 20:01:16
category:
- 实践
tags: 
- DNS
- dns-over-https
---
### 原由
今天莫名发现github打不开，但其他网站（国内）正常，查看域名解析，发现一直返回127.0.0.1，那怕我指定8.8.8.8解析依然存在问题。

![解析错误的截图](/images/202404/错误的解析.png)

但是当我指定使用TCP协议时却可以正常解析。

![TCP正常解析截图](/images/202404/TCP正常解析.png)

排查自己电脑设置发现一切正常，包括在live模式下依然存在问题。只能怀疑运营商搞的怪，尝试使用dns-over-https解决。

### 安装
```shell
sudo pacman -S dns-over-https
```
dns-over-https有俩个服务client和server，DNS解析请求从client通过https协议发送到server，由server代替请求域名解析服务，然后将响应通过https返回给client。

在这里我们只需要使用clinet，server使用阿里云的doh服务。

### 配置
修改/ect/resove.conf
```shell
nameserver 127.0.0.1

# 限制修改文件，防止NetworkManager改动配置
sudo chattr +i /etc/resolv.conf
```

修改client配置/etc/dns-over-https/doh-client.conf
```shell
# 添加这个阿里DoH服务器
[[upstream.upstream_ietf]]
    url = "https://dns.alidns.com/dns-query"
    weight = 50
```

### 启动服务
```shell
sudo systemctl start doh-client.service
```

