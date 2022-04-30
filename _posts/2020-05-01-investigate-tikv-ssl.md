---
layout: post
title: 网络问题排查的思路分析，以一次 TiKV 无法启用 TLS 为例
subtitle: 副标题
date: 2020-05-01
tags: [grpc, rust, tikv]
---

GitHub 社区有 bug 反馈在 mac 下为 tikv 集群开启 TLS 验证会导致连接 PD 错误，日志是

```
["PD failed to respond"] [err="Grpc(RpcFailure(RpcStatus { status: Unavailable, details: Some(\"Trying to connect an http1.x server\") }))"] [endpoints=127.0.0.1:2379]
```

## 调查

当时我看到的第一反应是不是 TLS 证书有问题，尝试下在我的 Linux 复现，没有成功，这个时候第一时间想到系统差异的问题，比如在 mac 下编译 tls 的库有问题。然后我找了同事的 mac 环境 ssh 上去测试了下并没出现这个问题，既然 issue 里给了 git hash，我决定编译一份一模一样的 tikv-server pd-server 去测试，成功在 ubuntu 20.04 下稳定复现。

稳定复现的问题排查起来就很简单，用 tcpdump 抓一下数据包，拿到 wireshark 分析后发现用老版本的 tidb 集群测试的时候，TLS 握手协议是 NPN，但在出问题的包中，看到 extension 是 ALPN。这里做了点功课，HTTP 协议是需要协商也确认 HTTP 版本。在开启 TLS 之后，会有一个叫 NPN(Next Protocol Negotiation) 的 TLS 扩展用于协商应用层该使用什么协议，后续 NPN 升级为 ALPN 协议，现在主流的 HTTPS 基本都是 ALPN 扩展了。

回到调查本身，查阅了 PD 代码，发现 TLS 部分主要用的官方库，因此和我用的新系统后下载了最新的 go version 编译的 PD 才有这个问题对应上了，我去询问了公司负责发版编译的同事，确认了他们用的 go 确实比较老，所以没有这个问题，然后就找到了 [commit](https://github.com/golang/go/commit/6da300b196df5fc3b33dd3bc87c477d46473abde)，[go/issues/28362](https://github.com/golang/go/issues/28362) 讨论为何要移除 NPN 的支持，是因为已经被 ALPN 取代因此直接删除也是合理的。

虽然新版的 go 强制走 ALPN 扩展，但为什么并没协商用 ALPN 呢，tikv 的部分也需要调查，在 grpc-sys/build.rs 中找到了有 PR 强制 disable ALPN，理由是因为老的 ubuntu 不支持（这也能 review 通过 🤣），剩下的工作只要强制开启 ALPN 并发一个大版本就解决了这个问题（破坏向后兼容），[grpc-rs/456](https://github.com/tikv/grpc-rs/pull/456)

## 排查工具

遇到没法直接定位的问题，基本都靠抓包，tcpdump 去抓数据，然后将数据拿到 mac or windows 的 wireshark 打开分析很方便，比如下图可以看到这个 Client Hello 就同时带有 NPN 和 ALPN 扩展。

![h2_tls_alpn_client](/images/2020-05-01-investigate-tikv-ssl.assets/h2_tls_alpn_client.png.webp)

## 参考

https://datatracker.ietf.org/doc/html/rfc7301

https://joji.me/en-us/blog/walkthrough-decrypt-ssl-tls-traffic-https-and-http2-in-wireshark/

https://imququ.com/post/protocol-negotiation-in-http2.html
