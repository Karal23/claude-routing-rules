# Claude 分流规则

依据 [Net.Coffee 的 Claude Code 分流页面](https://ip.net.coffee/claude/site.html)整理。**每个客户端只需订阅一份文件**，已合并核心域名、CDN、认证、监控、客服、NTP 分类和 IP 兜底；不需要再导入外部 geosite 订阅。

规则快照日期：2026-09-30。规则只决定匹配哪些流量，使用的代理节点/策略组由客户端配置指定。

| 客户端 | 完整规则订阅 | 说明 |
| --- | --- | --- |
| sing-box 1.15 系列 | [claude.srs](https://raw.githubusercontent.com/Karal23/claude-routing-rules/main/sing-box/claude.srs) | 推荐，已编译二进制 |
| sing-box JSON 源格式 | [claude.json](https://raw.githubusercontent.com/Karal23/claude-routing-rules/main/sing-box/claude.json) | 与 SRS 内容相同，二选一 |
| iOS Loon | [claude.list](https://raw.githubusercontent.com/Karal23/claude-routing-rules/main/loon/claude.list) | 原生远程规则，含 IPv4、IPv6 和 ASN |

**sing-box 导入**

将 [示例片段](examples/sing-box-route.json)中的条目合并到现有配置的 `route.rules` 和 `route.rule_set`，不要用它覆盖完整配置。把两处 `CLAUDE` 都改为现有代理出站或策略组的 tag：

```json
{
  "route": {
    "rules": [
      {
        "rule_set": "claude",
        "action": "route",
        "outbound": "CLAUDE"
      }
    ],
    "rule_set": [
      {
        "type": "remote",
        "tag": "claude",
        "format": "binary",
        "url": "https://raw.githubusercontent.com/Karal23/claude-routing-rules/main/sing-box/claude.srs",
        "http_client": { "detour": "CLAUDE" },
        "update_interval": "1d"
      }
    ]
  }
}
```

将这条路由放在必要的 DNS 劫持/嗅探规则之后、通用直连/地区/广告拦截规则之前。规则下载也通过 `CLAUDE` 出站。若使用 JSON 链接，将 `format` 改为 `source`，URL 改为上表中的 `.json` 链接。

推荐在现有配置中开启 `experimental.cache_file.enabled`，供远程规则缓存使用。DNS 解析、域名嗅探和 TUN 接管仍由现有配置负责，订阅规则本身不配置 DNS。

**Loon 导入**

将以下一行加入现有配置的 `[Remote Rule]`，或在 Loon 的订阅规则中添加上表的 `.list` URL，并选择指定代理策略。把 `CLAUDE` 改为已有节点或策略组名称：

```ini
[Remote Rule]
https://raw.githubusercontent.com/Karal23/claude-routing-rules/main/loon/claude.list, policy=CLAUDE, tag=Claude, enabled=true
```

将此订阅排在通用订阅之前。Loon 的本地规则和插件规则优先于订阅规则；若已有规则将相关域名直连或拦截，需要先调整冲突规则。IP 规则带 `no-resolve`，不会为了匹配这些 IP 规则额外解析域名；ASN 匹配使用 Loon 的 ASN 数据库。参见[官方优先级说明](https://nsloon.app/docs/Rule/)和[官方 IP 规则说明](https://nsloon.app/docs/Rule/ip_rule/)。

完整规则已包含 NTP 域名。若要转发相应 UDP 请求，还需在现有 `[General]` 中启用 `allow-udp-proxy = true`，并选择支持 UDP 的节点。可参考 [Loon 配置片段](examples/loon.conf)。

**覆盖范围与转换说明**

| 原文内容 | 本仓库处理 |
| --- | --- |
| Claude / Anthropic 核心域名、CDN、认证、监控、客服 | 保留精确域名、后缀与关键词语义 |
| `datadog`、`sentry`、`sift` | 三个关键词均纳入；补齐原文 JSON 示例漏写的 `sentry` |
| `geosite:anthropic` | 将固定版本的分类展开并合入同一文件 |
| `geosite:category-ntp` | 完整展开，包含中国和日本子分类、Apple、Google、Windows 等 NTP 域名 |
| `160.79.104.0/21`、`2607:6bc0::/32` | 两端均保留，IP 规则放在域名规则之后 |
| `AS399358` | Loon 使用原生 `IP-ASN`；sing-box 将公开已宣告前缀快照合入 `ip_cidr` |

sing-box 没有原生 ASN 匹配字段。此次 [RIPE RIS 前缀快照](sources/as399358.json)的三个前缀均已被原文的两个 CIDR 覆盖，因此去重后仍为两个 CIDR。这是**静态快照**，不等同于持续更新的 ASN 数据库；未来新增网段需更新快照并重新生成。

`anthropic.com` 的后缀规则已覆盖 `api`、`cdn`、`mcp`、`console`、`workbench` 等子域名。第三方关键词和 NTP 域名规则作用于设备所有匹配流量，因此也可能影响其他应用。没有加入“所有 UDP 123 端口”规则；直接以 IP 访问的 NTP 服务不一定命中域名分类。

NTP 使用 UTC 时间戳，代理 NTP 不会改变系统时区；原文关于 NTP 能返回代理所在地时区及其风控效果的说法不能据此保证。参见 [RFC 5905 第 6 节](https://www.rfc-editor.org/rfc/rfc5905.html#section-6)。本仓库实现的是规则转换，不保证账号风控结果。

**版本与验证**

截至 2026-09-30，[官方版本列表](https://github.com/SagerNet/sing-box/releases)尚无 `1.15.0` 正式版，1.15 系列最新为 **`1.15.0-alpha.9`**。本仓库按该版本验证；正式版发布后的实测不能提前承诺。源文件使用规则集格式版本 `2`，与客户端版本号是两个概念。

已使用官方 `1.15.0-alpha.9` Linux amd64 二进制完成编译、示例配置检查、424 次 JSON/SRS 原生匹配检查，以及通过本地 HTTP 入站和两个测试出站执行的 9 次实际路由检查（包括远程 SRS 下载加载、IPv4/IPv6 和未命中流量）。三个输出文件重复构建的 SHA-256 一致。Loon 的 139 条规则已检查语法和两端数据一致性；其中比 sing-box 多一条原生 ASN 规则。

Loon 按[官方规则示例](https://github.com/Loon0x00/LoonExampleConfig/blob/master/Rule/ExampleRule.list)及文档转换。构建环境没有 iOS Loon，无法完成真机导入和节点出口验证；请在 Loon 的请求记录中确认命中本订阅并使用预期策略。

**来源与重建**

- `sources/net-coffee.json`：从指定页面整理出的规则数据，包含正文和兜底章节的关键词。
- `sources/geosite/`：来自 [v2fly/domain-list-community](https://github.com/v2fly/domain-list-community/tree/2d71342eea1720da57261b55663c9d55619e9b1c)，固定提交 `2d71342eea1720da57261b55663c9d55619e9b1c`。保留[上游 MIT 许可证](sources/geosite/LICENSE)。
- `sources/as399358.json`：RIPEstat 的 AS399358 已宣告前缀及查询时间。
- sing-box 格式参考：[规则集源格式](https://sing-box.sagernet.org/configuration/rule-set/source-format/)、[远程规则集](https://sing-box.sagernet.org/configuration/rule-set/)。

安装 Python 3 和 sing-box 后，在仓库目录执行：

```sh
python3 scripts/build.py --sing-box /path/to/sing-box
sing-box rule-set match --format binary sing-box/claude.srs api.anthropic.com
sing-box rule-set match --format binary sing-box/claude.srs time.apple.com
sing-box rule-set match --format binary sing-box/claude.srs 160.79.104.1
```

生成过程只读取仓库内的来源快照，可复现且不需要联网。更新来源快照后重新运行脚本并提交三个输出文件即可。客户端定期下载的是本仓库最新提交，本仓库没有配置自动抓取上游内容的定时任务。
