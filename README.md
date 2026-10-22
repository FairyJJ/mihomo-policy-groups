# mihomo 分层代理策略组配置模板

基于 [mihomo](https://github.com/MetaCubeX/mihomo)（Clash.Meta 内核）的**分层代理策略组**配置模板：把「按场景分流 / 线路优选 / 按地区分流」拆成独立三层，规则路由与规则集单独成层。

提供**两个版本**，同一套骨架，差别只在「订阅源分档」这一层：

| 文件 | 订阅源 | 分档 | 地区组形态 | 适用场景 |
|---|---|---|---|---|
| [`config_聚合.yaml`](config_聚合.yaml) | 3 个：`premium` / `standard` / `backup` | 有 `[优]` / `[中]` / `[备]` | 入口组（select）→ 3 个档位组（url-test）+ 该地区全部节点 | 手上有多个机场/订阅，想按质量分档调度 |
| [`config_单一.yaml`](config_单一.yaml) | 1 个：`main` | 无 | **每地区两组**：`X自动`（url-test 择优）+ `X手动`（select 选点），都直接平铺该地区全部节点 | 只有一个订阅，或不需要分档，越平越好 |

> 所有订阅链接、个人域名、内网网段均已抹除为占位符，可直接公开分发；按下文「快速开始」替换为你自己的内容即可使用。
> 两种版本均通过 `mihomo -t`（v1.19.32）配置校验。

## 架构总览

```
rules（规则路由）
  └─ 命中 → 应用策略层（select，按场景选线路）
              └─ 线路特性层（url-test / fallback，全部节点自动测速）
                    └─ 地理策略层（按地区分组的 select / url-test 组）
                          └─ proxy-providers（订阅源）
```

| 层 | 策略组 | 类型 | 作用 |
|---|---|---|---|
| 应用策略层 | 网络测试 / ChatGPT / Gemini / Copilot / 其他AI / 电报消息 / 社交平台 / 游戏平台 / 货币平台 / 国际媒体 / TikTok / 苹果服务 / 谷歌服务 / 微软服务 / GitHub / 脸书服务 / 国外流量 / 国内流量 / 手动选择 | `select` | 按流量场景选择出口，成员平铺各地区组 + 线路特性组 |
| 线路特性层 | 自动选择 / 故障转移 | `url-test` / `fallback` | `use:` 直接拉取全部订阅节点，组内自动测速/故障切换，不引用任何策略组 |
| 地理策略层（两版都是 8 个地区） | 香港/台湾/日本/狮城/韩国/美国/欧洲/其他 | 聚合：`select` 入口 + `url-test` 档位；单一：`url-test` 自动 + `select` 手动 | 按正则筛出该地区节点，供应用层选用 |

地理策略层在两版里的内部结构不同：

```
聚合版                                单一版
香港智能 (select)                      香港自动 (url-test) ─ 档内自动择优
 ├─ 香港[优] (url-test) ← [优] 前缀     香港手动 (select)   ─ 该地区全部节点平铺
 ├─ 香港[中] (url-test) ← [中] 前缀
 ├─ 香港[备] (url-test) ← [备] 前缀
 └─ 该地区全部节点
```

单一版没有「智能」这种包装组：每个地区只有两个平级组，自动的走 `url-test` 自动择优，手动的走 `select` 手动选点，两者成员都是该地区全部节点（`include-all` + `filter`）。要手动指定某个节点，把上层场景组切到对应 `X手动` 再选节点即可。

## 核心设计

- **分档前缀（仅聚合版）**：三个订阅源分别通过 `override.additional-prefix` 给节点名追加 `[优] `/`[中] `/`[备] `，档位组用 `Filter<地区>_YOU/_ZHONG/_BEI` 正则按前缀收人；「优」档额外排除 `lite` 线路（`(?!.*(?i)lite)`）。单一版无前缀、无档位正则，节点名保持订阅原样。
- **「其他」地区**：`FilterOTHER` 为全部地区词表取反，收纳不属任何地区的残余节点，避免漏网。
- **地区组平铺**：聚合版 = 入口组 `proxies` 前置三个档位组 + `include-all` 该地区全部节点；单一版 = 每个地区两个平级组（`X自动` url-test / `X手动` select），只用 `include-all` + `filter`，无包装组、无子组。
- **无 一键智能 / 兜底流量**：两版均不设这两个聚合出口；`MATCH` 兜底落到 `国外流量`，`MIXED-AL` 监听端口同样指向 `国外流量`。
- **锚点模板**：`BaseProvider` / `BaseFB` / `BaseUT` / `OneSmart` / `SelectDC` / `SelectProxy` / `Filter*` 全部用 YAML anchor + merge key（`<<:`）定义，改一处生效全局。
- **规则顺序敏感点**（`rules` 内已注释标注）：
  - ChatGPT / Gemini / Copilot 必须位于 AI 兜底规则（`RULE-SET,AI`）之前；
  - TikTok 必须位于 Streaming（`category-entertainment`）之前；
  - GitHub 必须位于 微软服务 之前（`microsoft.mrs` 同样包含全部 github 域名）。
- **规则集来源**：全部取自 mihomo 官方 [MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat)（meta 分支 `.mrs`），经 **jsDelivr 镜像**分发，规避 `raw.githubusercontent.com` 不可达；`geox-url` 的 geoip/geosite/mmdb/asn 同样走 jsDelivr。
- **订阅拉取中转（聚合版）**：`standard` / `backup` 的 provider 带 `proxy: 订阅拉取`，由该 url-test 组（`use: [premium]`）挑一个 premium 节点去拉订阅。订阅域名被 TLS 阻断时直连会 `handshake timeout`，走节点中转才能拉到；premium 本身仍直连，不会成环。
- **分地区监听端口**：`listeners` 提供 50000–50007 八个 mixed 端口，按地区绑定出口，供浏览器插件 / 手机 APP 分地区访问。
- **fake-ip**：`fake-ip-filter` 放行 GeoCn/Private/China 规则集与小米/苹果域名，自建域名需自行追加。

## IPv6 策略

节点只有 IPv4，目标是：**代理链路一律 IPv4，国内域名仍保留 AAAA 供直连**。

| 位置 | 配置 | 作用 |
|---|---|---|
| `dns.fake-ip-range6` | **刻意不设置** | fake-ip 模式下 AAAA 直接空解析 —— 境外域名拿不到任何 IPv6 地址 |
| `dns.nameserver` / `dns.proxy-server-nameserver` | `#disable-ipv6=true` | 真实解析路径（白名单域名、节点域名、redir-host）丢弃 AAAA |
| `dns.nameserver-policy` | `rule-set:GeoCn` / `China` / `Private` → **不带参数**的国内 DoH | 国内 / 私有域名保留真实 AAAA，可 IPv6 直连 |
| `ipv6`（顶层） | `true` | 内核仍接受 IPv6，否则国内 AAAA 也白给 |

实测（mihomo v1.19.32，fake-ip 模式）：

```
AAAA www.google.com → 空             境外不下发 IPv6
A    www.google.com → 198.18.0.4     fake-ip 正常
AAAA www.baidu.com  → 2409:8c20:…    国内真实 AAAA（policy 放行）
A    www.baidu.com  → 223.109.82.…   国内真实 A
```

⚠️ 两个坑（都实测过）：

- **`dns.filter-aaaa` 不是 mihomo 的字段**。官方文档没有、二进制里没有，写上去不报错但**完全不生效**：加上它之后 `AAAA` 依旧返回 `fc00::4`。真正让境外 AAAA 变空的是「不配 `fake-ip-range6`」——一旦补上 range6，fake AAAA 立刻回来。
- `dns.ipv6: false` 有效但太粗：它是全局的，国内 AAAA 会一起消失，与「保留国内」冲突。

## 部署到 OpenClash（实测）

配置文件放进 `/etc/openclash/config/`，再把 `config_path` 指过去。**光放文件不够**——OpenClash 会用 uci 覆盖配置里的若干字段，下面这些开关必须一起改，否则配置会被"改回去"：

| uci 项 | 单一版（AX6 / arm64） | 聚合版（x86） | 为什么 |
|---|---|---|---|
| `openclash.config.ipv6_enable` | `1` | `1` | 为 `0` 时 OpenClash 把顶层 `ipv6` 写成 `false`，国内 AAAA 一起消失 |
| `openclash.config.ipv6_dns` | `1` | `1` | **它才是 `dns.ipv6` 的来源**；为 `0` 时强制 `dns.ipv6: false`，内核层直接不解析 AAAA |
| `openclash.config.auto_smart_switch` | `0` | `0` | 为 `1` 时把配置里所有 `url-test` 组改写成 `smart` 类型（面板显示 Smart），"自动"就不按延迟择优了 |
| `openclash.config.enable_custom_dns` | `0` | `0` | 为 `1` 时用界面里的 DNS 服务器列表覆盖配置文件的 `dns` 段，本项目的 AAAA 分流全部失效 |

改完必须**强制全量重建**，否则 OpenClash 的「快速启动」会继续用旧产物：

```sh
rm -f /tmp/openclash.change          # 清掉 mtime 记录，强制 QUICK_START=false
/etc/init.d/openclash stop
/etc/init.d/openclash start          # 重建 → /etc/openclash/<同名>.yaml → 内核 -f 该文件
```

⚠️ 三个坑：

- **`restart` 会空转**：init 脚本对已在运行的实例直接回 `OpenClash Already Start!`，配置根本不重建，得 `stop` + `start` 分开执行。
- **内核实际读的是 `/etc/openclash/<文件名>.yaml`**（`basename config_path` 的副本，经过 `yml_change.sh` 改写），不是 `config/` 下那份源文件；后台 `ps` 里能看到 `-f` 的真实路径。
- **别用 `exec /etc/init.d/openclash stop; ... start`**：`exec` 会替换掉当前 shell，后半段永不执行，结果只停不起。

部署后自检（面板端口 + secret 换成你的）：

```sh
curl -s -H "Authorization: Bearer <secret>" http://127.0.0.1:9090/proxies \
  | jq -r '.proxies | to_entries[] | select(.value.all) | "\(.key)\t\(.value.type)\t\(.value.all|length)"'
```

`type` 应该是 `URLTest` / `Selector` / `Fallback`，出现 `Smart` 就说明 `auto_smart_switch` 没关。

### PPPoE 下别额外配 `wan6`

`wan` 是 `proto pppoe` + `option ipv6 auto` 时，netifd 会自动建一个动态接口 **`wan_6`**（device = `pppoe-wan`，负责 DHCPv6 + PD）。如果 `network` 里还手写着一个 `config interface 'wan6'`（device `eth1`、proto `dhcpv6`），就成了**两个 DHCPv6 客户端**：

| 现象 | 原因 |
|---|---|
| LuCI 里多出一个 `wan6`，且一直不 up | `eth1` 上是 PPPoE 封装，这层根本没有 IPv6 可用 |
| 内网 IPv6 时通时断 / 完全不通 | 两个客户端各自向 ISP 要前缀，PD 来回换；`br-lan` 上同时挂着两个 /64 |
| 终端抓到已失效的前缀，IPv6 直接不通 | odhcpd 把前缀持久化在 `/tmp/odhcpd-piofolder/`，PD 换了旧条目还在，会继续通告 |

处理（`192.168.100.1` 实测）：

```sh
# 1) 删掉多余的那个，只留 netifd 自动建的 wan_6
uci delete network.wan6 && uci commit network
uci del_list firewall.@zone[1].network=wan6 && uci commit firewall
/etc/init.d/network reload        # reload 不会重拨 PPPoE，公网 IP 不变

# 2) 清前缀缓存，只通告当前 PD
rm -f /tmp/odhcpd-piofolder/*
/etc/init.d/odhcpd restart
```

验证：`ifstatus wan_6` 里应只有一个 `ipv6-prefix`；`ip -6 addr show br-lan` 只有一个 `2408:` 开头的 /64；`ip -6 neigh show dev br-lan` 不再出现旧前缀。

改前缀长度（`ip6assign`）或换光猫、重拨后，**记得清一次 PIO 缓存**——否则终端会带着旧前缀的地址继续用，表现就是"IPv6 挂了但路由器自己能上网"。

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
