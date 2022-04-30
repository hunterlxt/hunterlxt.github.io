---
layout: post
title: 为 TLS 实现零拷贝 sendfile
subtitle: 副标题
date: 2019-12-11
tags: [linux, network]
---

零拷贝主要指减少内核用户态的拷贝次数，一般来说没有特别的优化将一个文件从网卡发送出去，需要先读取文件内容到内核缓冲区，再从内核拷贝到用户态，用户态程序又系统调用 write 写入网络文件缓冲区，最后才会走 IO 流程发送出去。

## 现状

sendfile 系统调用是一种常用的零拷贝技术，但是现在的 openssl 1.1.1 中没有类似的接口，只有普通的写入，意味着势必会有用户态和内核态的转换。这是因为 openssl 是一个用户态的库，那自然而然加密工作需要在用户态完成，FaceBook 的研究表明他们服务器大概 2% 的开销都在拷贝上，而另外 10% 的开销用于加解密。因此他们向内核提供了一个 feature —— KTLS。即实现了一个 Linux 内核的 TLS socket，基本实现过程如下：

* Call connect() or accept() on a standard TCP file descriptor.
* A user space TLS library is used to complete a handshake.
We have tested with both GnuTLS and OpenSSL.
* Create a new KTLS socket file descriptor.
* Extract the TLS Initialization Vectors (IVs), session keys,
and sequence IDs from the TLS library. Use setsockopt on
the KTLS fd to pass them to the kernel.
* Use standard read(), write(), sendfile() and splice() system
calls on the KTLS fd.

## Demo

查了一下 openssl 1.1.1 还没有支持 KTLS，按照 [KTLS](https://www.kernel.org/doc/html/latest/networking/tls-offload.html) 文档，我们可以帮助 openssl 加入一个类似已经存在的 `SSL_write` 的接口，即 `SSL_sendfile`。这里需要按照 KTLS 文档做一个类似 SSL_enable_ktls 的接口以让 fd 和 ssl 信息在内核中启用，linux 内核的文档写的不清晰，需要对 iv key 等进行一次拷贝，因为握手的信息还是在用户态的，后面发送的时候就不会走多余的拷贝了。然后就可以包装一层 sendfile 作为 openssl 库用户的接口就可以跑起来了。

## 我的仓库

启用 KTLS 和 `SSL_sendfile` : [hunterlxt/openssl](https://github.com/hunterlxt/openssl/tree/TXXT/ktls-over-1_1_1)

client server demo: [hunterlxt/KTLS-Demo](https://github.com/hunterlxt/KTLS-Demo)

