# mihomo 分层代理策略组配置模板

基于 [mihomo](https://github.com/MetaCubeX/mihomo)（Clash.Meta 内核）的**分层代理策略组**配置模板：把「按场景分流 / 线路优选 / 按地区分流」拆成独立三层，规则路由与规则集单独成层。

提供**两个版本**，同一套骨架，差别只在「订阅源分档」这一层：

| 文件 | 订阅源 | 分档 | 地区组形态 | 适用场景 |
|---|---|---|---|---|
| [`config_聚合.yaml`](config_聚合.yaml) | 3 个：`premium` / `standard` / `backup` | 有 `[优]` / `[中]` / `[备]` | 入口组（select）→ 3 个档位组（url-test）+ 该地区全部节点 | 手上有多个机场/订阅，想按质量分档调度 |
| [`config_单一.yaml`](config_单一.yaml) | 1 个：`main` | 无 | 入口组（select）直接平铺该地区全部节点，无子组 | 只有一个订阅，或不需要分档，越平越好 |

> 所有订阅链接、个人域名、内网网段均已抹除为占位符，可直接公开分发；按下文「快速开始」替换为你自己的内容即可使用。
> 两种版本均通过 `mihomo -t`（v1.19.32）配置校验。

## 架构总览

```
rules（规则路由）
  └─ 命中 → 应用策略层（select，按场景选线路）
              └─ 线路特性层（url-test / fallback，全部节点自动测速）
                    └─ 地理策略层（入口组 select → 地区分组）
                          └─ proxy-providers（订阅源）
```

| 层 | 策略组 | 类型 | 作用 |
|---|---|---|---|
| 应用策略层 | 一键智能 / ChatGPT / Gemini / Copilot / 其他AI / 电报消息 / 社交平台 / 游戏平台 / 货币平台 / 国际媒体 / TikTok / 苹果服务 / 谷歌服务 / 微软服务 / GitHub / 脸书服务 / 国外流量 / 国内流量 / 兜底流量 / 手动选择 | `select` | 按流量场景选择出口，成员平铺各「地区智能」入口组 + 线路特性组 |
| 线路特性层 | 自动选择 / 故障转移 | `url-test` / `fallback` | `use:` 直接拉取全部订阅节点，组内自动测速/故障切换，不引用任何策略组 |
| 地理策略层（两版共用 8 个入口组） | 香港/台湾/日本/狮城/韩国/美国/欧洲/其他 智能 | `select` | 按正则筛出该地区节点，供应用层选用 |

地理策略层在两版里的内部结构不同：

```
聚合版                              单一版
香港智能 (select)                    香港智能 (select)
 ├─ 香港[优]  (url-test)  ← [优] 前缀   └─ 该地区全部节点（平铺，无子组）
 ├─ 香港[中]  (url-test)  ← [中] 前缀
 ├─ 香港[备]  (url-test)  ← [备] 前缀
 └─ 该地区全部节点
```

单一版若想「地区内自动择优」而不是手动选点，把该组 `type: select` 换成 `type: url-test` 即可（其余字段不用动）。

## 核心设计

- **分档前缀（仅聚合版）**：三个订阅源分别通过 `override.additional-prefix` 给节点名追加 `[优] `/`[中] `/`[备] `，档位组用 `Filter<地区>_YOU/_ZHONG/_BEI` 正则按前缀收人；「优」档额外排除 `lite` 线路（`(?!.*(?i)lite)`）。单一版无前缀、无档位正则，节点名保持订阅原样。
- **「其他」地区**：`FilterOTHER` 为全部地区词表取反，收纳不属任何地区的残余节点，避免漏网。
- **入口组平铺**：聚合版 = `proxies` 前置三个档位组 + `include-all` 该地区全部节点；单一版 = 仅 `include-all` + `filter`，选具体节点无需再进子组。
- **锚点模板**：`BaseProvider` / `BaseFB` / `BaseUT` / `SelectAL` / `OneSmart` / `SelectDC` / `SelectProxy` / `Filter*` 全部用 YAML anchor + merge key（`<<:`）定义，改一处生效全局。
- **规则顺序敏感点**（`rules` 内已注释标注）：
  - ChatGPT / Gemini / Copilot 必须位于 AI 兜底规则（`RULE-SET,AI`）之前；
  - TikTok 必须位于 Streaming（`category-entertainment`）之前；
  - GitHub 必须位于 微软服务 之前（`microsoft.mrs` 同样包含全部 github 域名）。
- **规则集来源**：全部取自 mihomo 官方 [MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat)（meta 分支 `.mrs`），经 **jsDelivr 镜像**分发，规避 `raw.githubusercontent.com` 不可达；`geox-url` 的 geoip/geosite/mmdb/asn 同样走 jsDelivr。
- **分地区监听端口**：`listeners` 提供 50000–50007 八个 mixed 端口，按地区绑定出口，供浏览器插件 / 手机 APP 分地区访问。
- **fake-ip**：`fake-ip-filter` 放行 GeoCn/Private/China 规则集与小米/苹果域名，自建域名需自行追加。

## 快速开始

### 用聚合版（多段订阅 + 分档）

1. **替换订阅**：`proxy-providers` 下 `premium` / `standard` / `backup` 三个 `url:` 换成你自己的订阅链接（保持 `[优]`/`[中]`/`[备]` 前缀逻辑，或按需修改档位筛选正则）。

### 用单一版（一段订阅 + 无分档）

1. **替换订阅**：只改 `proxy-providers` 下 `main` 的一条 `url:`，无需任何前缀。

### 两版通用

2. **调整端口**：`port` / `socks-port` / `mixed-port` 及 `listeners` 的 50000+ 端口，避免与本机其他代理冲突。
3. **调整内网网段**：`skip-auth-prefixes` 改为你的实际网段。
4. **追加自有域名**：`rules` 顶部与 `dns.fake-ip-filter` 中各加一行你的域名（模板中已留注释位）。
5. **节点排除关键词**：所有 `Filter*` 正则里的 `排除1|排除2` 是占位词，替换为你自己的节点名排除关键词（不需要可整段删除）。
6. **面板（可选）**：如需 zashboard 面板，取消注释 `external-controller` 等配置并下载 [zashboard](https://github.com/Zephyruso/zashboard) release 到 `ui` 目录。

## 校验

```bash
# 语法 + 逻辑校验（示例：单一版）
mihomo -t -d . -f config_单一.yaml
# → configuration file ... test is successful

# 两版差异（应只剩分档相关的删除）
diff config_聚合.yaml config_单一.yaml
```

## 文件说明

| 文件 | 说明 |
|---|---|
| `config_聚合.yaml` | 三订阅源 + `[优]`/`[中]`/`[备]` 分档版完整配置 |
| `config_单一.yaml` | 单订阅源、无分档的扁平化完整配置 |
| `LICENSE` | MIT |

## 致谢

- [mihomo](https://github.com/MetaCubeX/mihomo) — Clash.Meta 内核
- [MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat) — 规则集与 geo 数据
- [Koolson/Qure](https://github.com/Koolson/Qure) — 策略组图标集
- [666OS/YYDS](https://github.com/666OS/YYDS) — 部分图标
