---
layout: post
title: "自己动手做一个 CN+MY 的 GeoIP 数据库"
date: 2026-11-18
tags: [Surge, GeoIP, 折腾]
---

## 起因

我的 Surge 分流规则里有 `GEOIP,CN` 这一条，用来判断目标 IP 是不是在中国。Surge 默认用的是 MaxMind 的 GeoLite2-Country 数据库——全球版，几十 MB，更新还得挂代理慢慢下。

但我的需求其实很窄：我人在马来西亚，只需要分清两个国家——中国（CN）和马来西亚（MY）。马来西亚模板里我还写了一条 `GEOIP,MY,DIRECT`，保证本地流量永远直连。

那就自己做一个只含 CN+MY 的库好了。

## 数据从哪来

国家级的 IP 段清单是公开的，我用的是 Loyalsoldier/geoip 项目维护的 CIDR 列表：

- 中国（CN）：9,640 个网段
- 马来西亚（MY）：3,989 个网段
- IPv4、IPv6 都有

## 怎么做

.mmdb 是 MaxMind 的二进制数据库格式，不能拿文本编辑器糊弄。我写了一个小的 Go 程序，用 `mmdbwriter` 把 CIDR 列表逐条写进去，数据库类型声明为 `GeoLite2-Country`——这样 Surge 会把它当成官方库来读，不需要任何 hack。

最后产出的文件只有 **379 KB**，比官方全球库小了两个数量级。

## 验证

做完不能直接信，必须验：

1. 拿几个已知的 CN、MY、其他国家的 IP 逐个查询，返回的国家代码要对
2. CN/MY 以外的 IP 应该查无记录——这正是「纯名单库」该有的行为，和 chnroutes 那类库一致
3. 文件传到 GitHub 后，再从 raw 地址下载回来，和本地文件比对哈希，确认上传没损坏

三关都过了，我才把两个 Surge 模板的 `geoip-maxmind-url` 换成这个自建库的地址。

## 结果

现在我的配置里，`GEOIP,CN` 和 `GEOIP,MY` 共用这一个 379 KB 的文件，更新就是重新跑一遍构建脚本。库和构建方法都放在我的 Surge6-Rules 仓库里（`geoip/cn-my.mmdb`），有同样需求的可以直接拿去用。

> 后记：后来给 Loon 做模板时，又用同样思路自建了一个完整的 ASN 数据库（`geoip/asn.mmdb`，数据源是 BGP 路由表），那是另一篇的故事了。
