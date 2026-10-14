# mihomo 分层代理策略组配置模板

基于 [mihomo](https://github.com/MetaCubeX/mihomo)（Clash.Meta 内核）的**分层代理策略组**配置模板：把「按场景分流 / 线路优选 / 按地区分流」拆成独立三层，配合订阅节点的分档前缀（`[优]` / `[中]` / `[备]`）实现全自动分级调度，规则路由与规则集单独成层。

> 所有订阅链接、个人域名、内网网段均已抹除为占位符，可直接公开分发；按下文「快速开始」替换为你自己的内容即可使用。

## 架构总览

```
rules（规则路由）
  └─ 命中 → 应用策略层（select，按场景选线路）
              └─ 线路特性层（url-test / fallback，全部节点自动测速）
                    └─ 地理策略层（入口组 select → 地区[优]/[中]/[备] url-test 分组）
                          └─ proxy-providers（三个订阅源，按前缀分档）
```

| 层 | 策略组 | 类型 | 作用 |
|---|---|---|---|
| 应用策略层 | 一键智能 / ChatGPT / Gemini / Copilot / 其他AI / 电报消息 / 社交平台 / 游戏平台 / 货币平台 / 国际媒体 / TikTok / 苹果服务 / 谷歌服务 / 微软服务 / GitHub / 脸书服务 / 国外流量 / 国内流量 / 兜底流量 / 手动选择 | `select` | 按流量场景选择出口，成员平铺各「地区智能」入口组 + 线路特性组 |
| 线路特性层 | 自动选择 / 故障转移 | `url-test` / `fallback` | `use:` 直接拉取全部订阅节点，组内自动测速/故障切换，不引用任何策略组 |
| 地理策略层 | 香港/台湾/日本/狮城/韩国/美国/欧洲/其他 智能（入口组）+ 各地 `[优]`/`[中]`/`[备]`（档位组） | `select` + `url-test` | 入口组平铺「三档 + 该地区全部节点」；档位组按正则筛选，在档内自动测速 |

## 核心设计

- **分档前缀**：三个订阅源分别通过 `override.additional-prefix` 给节点名追加 `[优] `/`[中] `/`[备] `，档位组用 `Filter<地区>_YOU/_ZHONG/_BEI` 正则按前缀收人；「优」档额外排除 `lite` 线路（`(?!.*(?i)lite)`）。
- **「其他」地区**：`FilterOTHER` 为全部地区词表取反，收纳不属任何地区的残余节点，避免漏网。
- **入口组平铺**：地区入口组 = `include-all`（该地区全部节点）+ `proxies` 前置三个档位组，选具体节点无需再进子组。
- **锚点模板**：`BaseProvider` / `BaseFB` / `BaseUT` / `SelectAL` / `OneSmart` / `SelectDC` / `SelectProxy` / `Filter*` 全部用 YAML anchor + merge key（`<<:`）定义，改一处生效全局。
- **规则顺序敏感点**（`rules` 内已注释标注）：
  - ChatGPT / Gemini / Copilot 必须位于 AI 兜底规则（`RULE-SET,AI`）之前；
  - TikTok 必须位于 Streaming（`category-entertainment`）之前；
  - GitHub 必须位于 微软服务 之前（`microsoft.mrs` 同样包含全部 github 域名）。
- **规则集来源**：全部取自 mihomo 官方 [MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat)（meta 分支 `.mrs`），经 **jsDelivr 镜像**分发，规避 `raw.githubusercontent.com` 不可达；`geox-url` 的 geoip/geosite/mmdb/asn 同样走 jsDelivr。
- **分地区监听端口**：`listeners` 提供 50000–50007 八个 mixed 端口，按地区绑定出口，供浏览器插件 / 手机 APP 分地区访问。
- **fake-ip**：`fake-ip-filter` 放行 GeoCn/Private/China 规则集与小米/苹果域名，自建域名需自行追加。

## 快速开始

1. **替换订阅**：`proxy-providers` 下 `premium` / `standard` / `backup` 三个 `url:` 换成你自己的订阅链接（保持 `[优]`/`[中]`/`[备]` 前缀逻辑，或按需修改档位筛选正则）。
2. **调整端口**：`port` / `socks-port` / `mixed-port` 及 `listeners` 的 50000+ 端口，避免与本机其他代理冲突。
3. **调整内网网段**：`skip-auth-prefixes` 改为你的实际网段。
4. **追加自有域名**：`rules` 顶部与 `dns.fake-ip-filter` 中各加一行你的域名（模板中已留注释位）。
5. **节点排除关键词**：所有 `Filter*` 正则里的 `排除1|排除2` 是占位词，替换为你自己的节点名排除关键词（不需要可整段删除）。
6. **面板（可选）**：如需 zashboard 面板，取消注释 `external-controller` 等配置并下载 [zashboard](https://github.com/Zephyruso/zashboard) release 到 `ui` 目录。

## 文件说明

| 文件 | 说明 |
|---|---|
| `config.yaml` | 完整 mihomo 配置（锚点 / 提供者 / 监听器 / 策略组 / 规则 / 规则集） |
| `LICENSE` | MIT |

## 致谢

- [mihomo](https://github.com/MetaCubeX/mihomo) — Clash.Meta 内核
- [MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat) — 规则集与 geo 数据
- [Koolson/Qure](https://github.com/Koolson/Qure) — 策略组图标集
- [666OS/YYDS](https://github.com/666OS/YYDS) — 部分图标
