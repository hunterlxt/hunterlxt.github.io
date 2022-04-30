---
layout: post
title: WireGuard：随时随地一键回家
subtitle: 副标题
date: 2021-11-13
tags: [linux, network]
---

双十一薅了腾讯云三年vps的羊毛，准备拿来部署一个局域网方便在外用家里的开发机。wireguard ubuntu 原生的 VPN 工具，不过因为协议没有混淆，部署在海外 VPS 很容易被封，建议只用于正常 VPN 需求。部署很简单只需要： 1.公钥/私钥 2.wg配置文件 即可。

## 安装

```
sudo apt update && sudo apt install wireguard
```

## 服务端

首先打开云主机的 12345 端口（wg server通信用）

生成密钥

```
$ wg genkey | sudo tee /etc/wireguard/privatekey | wg pubkey | sudo tee /etc/wireguard/publickey
$ sudo vim /etc/wireguard/wg0.conf
```

配置

```
# wg0.conf
# PostUp 的设置是为了让 peer 的非内部 ip 段流量可以走 eth0，即实现全局代理，不加这个脚本只能实现基本的局域网
# Address 任意一个内网段地址都行

[Interface]
Address = 192.168.99.1/24
ListenPort = 12345
PrivateKey = 
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```

```
sudo wg set wg0 peer <公钥> allowed-ips 192.168.99.2
```

转发设置

```
sudo vim /etc/sysctl.conf
net.ipv4.ip_forward = 1
sudo sysctl -p
```

生效

```
sudo wg-quick up wg0
```

开机自启

```
sudo systemctl enable wg-quick@wg0
```

## 客户端

配置

```
# AllowedIPs可以直接设置0.0.0.0即全局流量走wg，当前例子里所有99.x的请求走wg

[Interface]
PrivateKey = 
Address = 192.168.99.2/24

[Peer]
PublicKey = 
AllowedIPs = 192.168.99.0/24
Endpoint = <服务器公网IP>:12345
PersistentKeepalive = 15
```

生效

```
sudo wg-quick up wg0
```

开机自启

```
sudo systemctl enable wg-quick@wg0
```

## MacOs

需要注意 mac etc 目录

```
sudo mkdir -p /usr/local/etc/wireguard
cd wireguard
umask 077
wg genkey | tee privatekey | wg pubkey > publicke
# 剩下配置方法和 Linux 一致
```

## Windows

Download: https://download.wireguard.com/windows-client/wireguard-installer.exe
