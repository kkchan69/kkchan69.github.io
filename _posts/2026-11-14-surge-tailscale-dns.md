---
layout: post
title: "Surge 内置 Tailscale 踩坑：DNS lookup failed 的解法"
date: 2026-11-14
tags: [Surge, Tailscale, 踩坑]
---

## 想干什么

我想让 Surge 的 `FINAL` 流量从我自己的一台 VPS 出去。手边正好有 Tailscale，而 Surge 内置了 Tailscale 策略——用内置的好处是避开 iOS 的「同时只能开一个 VPN」限制，不用在 Surge 和 Tailscale App 之间二选一。

流程本来很顺：Surge 里开交互登录，VPS 上装 Tailscale 并声明为出口节点（exit node），控制台里批准，然后在 Surge 的 `[Tailscale]` 段里写上 `exit-node`。结果策略测试直接报错：

```
DNS lookup failed: No available upstream DNS server
```

## 病根

我的 tailnet 只开了 MagicDNS，**没有配全局 nameserver**。

平时用 Tailscale App 时不觉得，因为 App 会自己处理解析。但 Surge 内嵌的这个 Tailscale 节点要先把出口节点的 MagicDNS 名字解析成 Tailscale 内网 IP 才能连上去——它手里没有任何上游 DNS 可问，名字解不出来，连接根本起不来。

## 解法

在 `[Tailscale]` 段里给这个节点单独指定一个 DNS 服务器就行：

```
[Tailscale My-Tailnet]
interactive-login = true
exit-node = <你的出口节点名字>
dns-server = 1.1.1.1
```

加上 `dns-server = 1.1.1.1` 这一行，策略测试立刻通过，FINAL 从此走 VPS 出口。这个设置只作用于 Tailscale 节点本身，不影响 Surge 其他部分的 DNS 配置，这也是它比「去 tailnet 后台配全局 nameserver」更干净的地方。

## 顺带两个坑

**段名不要改。** `[Tailscale xxxxxxxx]` 里那串随机字符是每台设备自己生成的，登录状态就绑在段名上。我有几台设备，每台的段名都不一样——千万别为了「整齐」把另一台的段名抄过来，抄完就得重新登录。

**每台设备单独登录一遍。** 配置文件可以同步，登录态不行。新设备上把 Surge 配置导过去之后，Tailscale 段还是要在那台设备上重新走一遍交互登录，然后单独补上 `exit-node` 和 `dns-server` 两行。
