---
title: WIFI热点无效问题
date: 2024-05-07 13:26:23
category:
- 实践
tags: 
- network
- wifi
- docker
---
因为拉的宽带质量不是很稳定，所以改成使用手机卡通过电脑共享WIFI网络给手机上网，Plasma6上使用NetworkManager可以很方便的设置开启共享网络。
![共享网络设置](/images/202405/hotspot设置.png)

开启热点后，NetworkManager会自动生成一下防火墙规则，用于转发网络请求。
```shell
table ip nm-shared-wlp3s0 {
    # 源地址转换
	chain nat_postrouting {
		type nat hook postrouting priority srcnat; policy accept;
		ip saddr 10.42.0.0/24 ip daddr != 10.42.0.0/24 masquerade
	}

	chain filter_forward {
		type filter hook forward priority filter; policy accept;
		ip daddr 10.42.0.0/24 oifname "wlp3s0" ct state { established, related } accept # 允许回复包
		ip saddr 10.42.0.0/24 iifname "wlp3s0" accept # 允许热点内部主机新连接
		iifname "wlp3s0" oifname "wlp3s0" accept # 允许热点内设备互通
		iifname "wlp3s0" reject
		oifname "wlp3s0" reject
	}
}
```

当启动docker服务后，会自动添加一下规则
```shell
chain FORWARD {
	type filter hook forward priority filter; policy drop; # 默认策略是drop
	counter packets 178 bytes 20547 jump DOCKER-USER
	counter packets 178 bytes 20547 jump DOCKER-FORWARD
}

chain DOCKER-USER {
}

chain DOCKER-FORWARD {
	counter packets 178 bytes 20547 jump DOCKER-CT
	counter packets 178 bytes 20547 jump DOCKER-ISOLATION-STAGE-1
	counter packets 178 bytes 20547 jump DOCKER-BRIDGE
	iifname "docker0" counter packets 0 bytes 0 accept
}

chain DOCKER-CT {
	oifname "docker0" xt match "conntrack" counter packets 0 bytes 0 accept
}

chain DOCKER-ISOLATION-STAGE-1 {
	iifname "docker0" oifname != "docker0" counter packets 0 bytes 0 jump DOCKER-ISOLATION-STAGE-2
}

chain DOCKER-ISOLATION-STAGE-2 {
	oifname "docker0" counter packets 0 bytes 0 drop
}

chain DOCKER-BRIDGE {
	oifname "docker0" counter packets 0 bytes 0 jump DOCKER
}
```
可以看到FORWARD链默认策略是drop，且其规则优先级和NetworkManger生成的一样，属于谁先注册谁先生效，导致最终从wlp3s0网口进来的流量都丢弃了；手机上看到的就是连得上WIFI，但是无法访问外网。

这里简单处理，在DOCKER-USER链添加规则，放行wlp3s0网口流量（不是最好的方法）

添加以下文件`/etc/systemd/system/docker.service.d/add-iptables-rules.conf`，该文件设定在docker服务启动后指定添加规则的命令。
```shell
[Service]
ExecStartPost=/usr/bin/bash -c 'sleep 5 && sudo iptables -I DOCKER-USER -i wlp3s0 -j ACCEPT && sudo iptables -I DOCKER-USER -o wlp3s0 -j ACCEPT'
```

--------------------
[arch wiki](https://wiki.archlinuxcn.org/wiki/Nftables#%E4%B8%8E_Docker_%E4%B8%80%E8%B5%B7%E5%B7%A5%E4%BD%9C)上提供了一个其他思路。


1.为docker服务开启新的命令空间
新增文件`/etc/systemd/system/docker.service.d/netns.conf`
```shell
[Service]
# 开启新网络命令空间
# 接下来的ExecStartPre和ExecStart命令都在会这个命令空间下执行
PrivateNetwork=yes
# PrivateNetwork=yes会同时将PrivateMounts设置为yes,这里手动恢复
PrivateMounts=No

# cleanup
ExecStartPre=-nsenter -t 1 -n -- ip link delete docker0

# add veth
ExecStartPre=nsenter -t 1 -n -- ip link add docker0 type veth peer name docker0_ns
ExecStartPre=sh -c 'nsenter -t 1 -n -- ip link set docker0_ns netns "$$BASHPID" && true'
ExecStartPre=ip link set docker0_ns name eth0

# bring host online
ExecStartPre=nsenter -t 1 -n -- ip addr add 10.0.0.1/24 dev docker0
ExecStartPre=nsenter -t 1 -n -- ip link set docker0 up

# bring ns online
ExecStartPre=ip addr add 10.0.0.100/24 dev eth0
ExecStartPre=ip link set eth0 up
ExecStartPre=ip route add default via 10.0.0.1 dev eth0
```

2.配置防火墙和转发规则
```shell
sudo nft add table ip docker_nat
sudo nft add chain ip docker_nat postrouting '{ type nat hook postrouting priority srcnat; }'
# masquerade规则，enp197s0f3u4是我的外网网卡
sudo nft add rule ip docker_nat postrouting ip saddr 10.0.0.0/24 oif "enp197s0f3u4" masquerade

# 为docker0接口开启网络转发
sudo sysctl -w net.ipv4.conf.docker0.forwarding=1
```

完整脚本
```shell
[Service]
PrivateNetwork=yes
PrivateMounts=No

# cleanup
ExecStartPre=-nsenter -t 1 -n -- ip link delete docker0

# add veth
ExecStartPre=nsenter -t 1 -n -- ip link add docker0 type veth peer name docker0_ns
ExecStartPre=sh -c 'nsenter -t 1 -n -- ip link set docker0_ns netns "$$BASHPID" && true'
ExecStartPre=ip link set docker0_ns name eth0

# bring host online
ExecStartPre=nsenter -t 1 -n -- ip addr add 10.0.0.1/24 dev docker0
ExecStartPre=nsenter -t 1 -n -- ip link set docker0 up

# bring ns online
ExecStartPre=ip addr add 10.0.0.100/24 dev eth0
ExecStartPre=ip link set eth0 up
ExecStartPre=ip route add default via 10.0.0.1 dev eth0

ExecStartPre=-nsenter -t 1 -n -- nft delete table docker_nat
ExecStartPre=nsenter -t 1 -n -- nft add table ip docker_nat
ExecStartPre=nsenter -t 1 -n -- nft add chain ip docker_nat postrouting '{ type nat hook postrouting priority srcnat; }'
ExecStartPre=nsenter -t 1 -n -- nft add rule ip docker_nat postrouting ip saddr 10.0.0.0/24 oif "enp197s0f3u4" masquerade

ExecStartPost=nsenter -t 1 -n -- sysctl -w net.ipv4.conf.docker0.forwarding=1
```