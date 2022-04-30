---
layout: post
title: 实践分布式系统下的自动故障处理
subtitle: 副标题
date: 2022-01-10
tags: [raft, tikv]
---

分布式系统的一个优点就是具备优秀的可用性，但是现在的 tikv-server 遇到文件损坏或者 IO 错误时会直接 panic，导致整个系统的延迟短暂上涨，而如果像去年某客户遇到的整批采购的多个硬盘都出现了坏道的问题，那就更惨了，丢部分数据已经算小事，如何快速恢复启动这样的多节点损坏的集群，想想都头大 😭

## 取舍

去年读的[论文](https://hunterlxt.github.io/recovery-for-consensus)对分布式系统下的故障恢复进行了讨论并提出了一些设计，但是结合在实际工程看，有些设计比较复杂，实际做到利用多副本恢复可以取一个 trade off。比如论文提到将 local meta data 多存一份，这个感觉就没必要了，也没人保证 2 个盘的 meta data 就不会同时丢失，而且根据故障局部性原理，一般发生故障的时候都是一批盘或者一个机架或者机房的问题，如果跨 AZ 存储，延迟的问题又会出现。根据 2/8 原理，只考虑恢复 applied 的数据就能满足大部分场景的需要。

TiKV 使用的是 multi-raft 管理数据，即将数据按照 range 切割成多个 region，每一个 region 是一个独立的 raft group。根据 2/8 原理，设计不考虑恢复所有的情况，而 applied 的数据会存储在 kv db 中。kv db 在 tikv 里是由一个 RocksDB 实例承担。所以这个想利用分布式做在线无感知的恢复，实则对 RocksDB 的故障错误进行检测，并从其他副本复制数据，最后对损坏的文件进行删除即可恢复。

## 挑战

**检测错误：**RocksDB 通过 event listener 允许给使用者一定的 hook 能力，当处理到背景错误时，我们改造一下它的 bg_error 并将其清空，否则 Corruption 和 IO error 都是 FATAL 级别的，会直接让 RocksDB 拒绝任何读写请求。这样处理后正常的读写依然可以继续执行，至少我们暂时保住了可用性。

**调度**：将故障 SST 的 range 信息比对 meta 信息，可以算出来这个 range 重叠的 regions，在 store heartbeat 中将其上报给 PD 节点（负责调度 tikv 集群，保存 tikv 集群的元信息，负责给 client 提供路由）。PD 在检测到某个 tikv 节点出现了损坏的 regions 后立刻对其执行 evict leaders 操作，这是一个防御性操作，以防止该节点出现更为严重的存储层错误导致集群的读写收到影响。这以后将该 tikv 节点对应 region 的所持有的副本下发删除命令。该 tikv 节点执行删除后，rocksdb 持有的文件不一定真的删除。因为 DeleteRange 并不会立刻删除掉 range 覆盖的 sst 文件，这个操作会在后续的 compaction 时完成，但问题来了，当后续 compaction 读取到故障文件会再次报错，因此这里需要用 DeleteFilesInRange 接口删除该文件的 range，如果 range 覆盖，此接口会立刻删除 sst 文件而不读取。

**正确性保证**：删除文件的时候该 tikv 节点已经没有对应 region（range）的信息，直接对存储引擎操作是安全的。另外一个比较复杂的正确性证明是 add back/split/merge 的时候，这段 range 会不会又和新的 region 重叠，这里可以通过 mutex 保护 region meta 信息来解决。

当然操作的时候存在一些实现问题，这个工作还在进行中，如果有想法在 [issue](https://github.com/tikv/tikv/issues/10578) 里见

## 块设备层故障模拟

写坏文件的测试用脚本即可，但是 IO 层的注入可以考虑用 Device Mapper，这个工具可以加载一些自定义的 policy 到逻辑和实际设备的映射中。每一个 bio 都会参考这个映射逻辑。可以实现从加载 table 时间开 始，设备可用 up interval 秒，然后在 down interval 秒内表现出不可靠的行为 ，然后重复此循环。policy 只会返回 3 种结果: HIT、MISS、issue migration。

```shell
// 创建一个简单的 mapper，在 down 周期内 IO 均返回 error
dmsetup create test-dm --table '0 1024 flakey /dev/nvme0n1 0 5 1'
// 对部分字节修改实现 Corruption 的效果
dmsetup create test-dm --table '0 800 flakey /dev/sdb 0 4 4 5 corrupt_bio_byte 128 r 1 0'
```

局限性：只能对挂载的整个目录进行注入，无法稳定复现，稳定复现还需要依赖脚本检测和集成测试。

参考：https://www.kernel.org/doc/html/latest/admin-guide/device-mapper/dm-flakey.html#
