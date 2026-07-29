# Clash 配置与协议速查手册 2026

> 面向已经买了机场、但卡在「怎么配、为什么不通」的人。
> 协议怎么选、规则怎么写、报错怎么查——**给可复制的配置片段和判断方法，不给测速数字**。

内容维护：[clashvpnss.com](https://clashvpnss.com/) · 更新于 2026-07-25 · 欢迎提 Issue 纠错

---

## 这份手册解决什么

| 你的问题 | 跳转 |
| --- | --- |
| 这问题是配置错了，还是机场本身的限制？ | [先分清责任边界](#零配置之前先分清哪些是配置改不了的) |
| 协议这么多，我该用哪个？ | [协议对比总表](#一协议对比选哪个) |
| 配置文件那一堆字段什么意思？ | [Clash 配置结构](#二clash-配置结构) |
| 为什么我写的规则不生效？ | [规则优先级陷阱](#三分流规则实战) |
| 显示已连接，但就是打不开网页 | [故障排查决策树](#五故障排查决策树) |
| 机场说是专线，怎么验证？ | [自己验证的方法](#六自己验证的方法) |
| Windows / Mac / iOS / 安卓用哪个客户端？ | [客户端选型](#四客户端选型分平台) |

**不解决**：哪家机场好、多少钱、速度多快。那属于选购问题，本手册只管配置和原理。

---

## 零、配置之前：先分清哪些是配置改不了的

调配置之前先花两分钟确认这件事，能省掉大量无效折腾——**很多人反复改规则、换客户端，但问题的根源在服务端，改什么都没用。**

### 这些由机场决定，客户端改不了

| 能力 | 由谁决定 | 影响什么 |
| --- | --- | --- |
| **支持哪些协议** | 机场服务端 | 决定你能不能用 Hysteria 2、VLESS-Reality 等新方案 |
| **是否支持 UDP 转发** | 机场服务端 | 游戏联机、语音通话能否正常 |
| **NAT 类型**（FullCone / Symmetric） | 机场服务端 | P2P 联机能否打洞成功 |
| **线路类型**（IPLC 专线 / 公网中转） | 机场采购 | 晚高峰是否拥堵——这是最影响体感的一项 |
| **出口 IP 属性**（原生 / 机房 IP） | 机场采购 | 流媒体与 AI 服务能否解锁 |
| **流量倍率** | 机场计费策略 | 同样看一部片子扣多少流量 |

### 判断表：这个问题改配置有用吗？

| 症状 | 根源 | 改配置能解决吗 |
| --- | --- | --- |
| 节点全绿但打不开网页 | 本地系统代理 / 系统时间 | ✅ 能 |
| 规则写了却不生效 | 规则顺序错误 | ✅ 能 |
| 国内网站变慢 | 开了全局模式 | ✅ 能 |
| 某些程序不走代理 | 未开 TUN 模式 | ✅ 能 |
| 白天正常、晚上卡顿 | 线路走公网国际出口 | ❌ 换线路类型 |
| 游戏能登录进不了对战 | 链路不支持 UDP 转发 | ❌ 换机场或换节点 |
| Netflix 提示仅限自制剧 | 出口 IP 属性不合格 | ❌ 换节点/换机场 |
| AI 工具持续风控 | IP 信誉被标记 | ⚠️ 部分能（固定出口分流），但根源在 IP |
| 组队失败、NAT4 | NAT 类型受限 | ❌ 服务端决定 |

**规律很清晰**：本地行为（代理开关、规则、模式、TUN）能改；线路质量与 IP 属性改不了。

### 怎么快速看清你的机场给了什么

1. **看订阅里的节点信息**——客户端里节点的协议类型直接可见（`vless` / `hysteria2` / `ss` 等），这就是你能用的范围。
2. **看后台套餐说明**——流量倍率、是否标称专线、设备数限制通常写在这里。
3. **自己验证宣称**——标称专线可以用 `mtr` 查路由路径，方法见[第六节](#六自己验证的方法)。厂商写「高速稳定」这类词没有统一标准，不能作为比较依据。

> 如果确认是线路或 IP 属性的问题，那属于选购范畴，本手册不做推荐。
> 判断方法（线路类型差异、计费陷阱、跑路风险自查）见 [机场怎么选](https://clashvpnss.com/airport.html)；
> 专线真伪的验证原理见 [IPLC 与 IEPL 是什么](https://clashvpnss.com/airport_article_4.html)。

---

## 一、协议对比：选哪个

### 总表

| 协议 | 传输层 | 核心机制 | 抗检测思路 | 适用场景 |
| --- | --- | --- | --- | --- |
| **Shadowsocks** | TCP/UDP | 预共享密钥 + AEAD 加密 | 无协议特征（全随机字节） | 链路干净、要求低开销；移动端省电好 |
| **VMess** | TCP | UUID + 时间戳认证 | 可叠加 WS/gRPC/H2 + TLS | 需要 CDN 中转的场景（已逐步被 VLESS 取代） |
| **VLESS** | TCP | 无内层加密（依赖外层 TLS） | 头部轻量，配合 Reality | 当前主流；新部署首选 |
| **Trojan** | TCP | 真 TLS + SHA-224 密码 | 认证失败回退真实网站 | 对抗主动探测 |
| **Hysteria 2** | UDP (QUIC) | Brutal 拥塞控制 + 多路复用 | Salamander 混淆 | 链路丢包严重、要吞吐量 |
| **Reality** | TCP | X25519 密钥交换，借用第三方证书 | 握手指向真实大站，无需自备证书 | 需要规避证书登记痕迹 |

### 按场景决策

```
链路丢包严重（>5%）、要下大文件/看高码率视频
  └→ Hysteria 2（UDP 系，不因丢包降速）
     ⚠ 但部分网络对非常规 UDP 端口有 QoS 或封禁，需实测

主要担心被主动探测识别
  └→ VLESS-Reality（借用真实站点证书）或 Trojan（回退真实网站）

需要走 CDN / 443 端口伪装成正常 Web 流量
  └→ VLESS + WebSocket + TLS

链路本身没有针对性干扰，只要一条稳定加密通道
  └→ Shadowsocks + AEAD（开销最低，手机功耗表现最好）
```

**务实建议**：手上保留**两种不同传输层**的方案（一个 TCP 系如 VLESS-Reality，一个 UDP 系如 Hysteria 2）。不同的限制手段对不同传输层效果差异很大，同时被覆盖的概率低得多。

### 几个容易踩的认知误区

- **VMess 的 `alterId` 已废弃**，应统一设为 `0` 并使用 VMessAEAD。还在用非零 alterId 的是旧配置。
- **VMess 对系统时间敏感**：客户端与服务端时间偏差通常需在 90 秒内，否则握手直接失败。这是新手最常见的隐蔽报错来源。
- **Trojan 部署必须配一个真实可访问的网站**作为回退目标，否则回退时暴露的空白页反而成了新特征。
- **Salamander 是混淆不是加密**：安全性由 QUIC 的 TLS 层保证，它只负责改变流量外观。
- **没有任何协议提供永久保证**。这是持续对抗的过程，选型时更该关注方案是否仍在活跃维护。

> 原理详解：[协议抗封锁演进](https://clashvpnss.com/protocol_article_1.html) ·
> [Trojan 伪装机制](https://clashvpnss.com/protocol_article_4.html) ·
> [Hysteria 2 传输模型](https://clashvpnss.com/protocol_article_7.html) ·
> [SS 与 VMess 对比](https://clashvpnss.com/protocol_article_9.html) ·
> [Reality 深度剖析](https://clashvpnss.com/protocol_article_12.html)

---

## 二、Clash 配置结构

一份配置由这几块组成，**手动要改的一般只有 `proxy-groups` 和 `rules`**（`proxies` 通常由订阅自动填充）：

```yaml
port: 7890            # HTTP 代理端口
socks-port: 7891      # SOCKS5 代理端口
mode: rule            # rule / global / direct
log-level: info

proxies:              # 节点列表（订阅自动填充）
proxy-groups:         # 节点分组与选择策略
rules:                # 分流规则，自上而下匹配
```

### 三种模式的区别

| 模式 | 行为 | 什么时候用 |
| --- | --- | --- |
| `rule` | 按规则分流：国内直连、国外代理 | **日常就用这个** |
| `global` | 所有流量走代理（含国内） | 仅排查问题时临时用 |
| `direct` | 完全不走代理 | 等同于关闭 |

长期开 `global` 的三个代价：国内网站变慢（20ms 的站绕道海外变 200ms+）、白白消耗套餐流量、**银行/支付/政务类服务可能触发风控**。

### proxy-groups：三种策略

```yaml
proxy-groups:
  - name: PROXY
    type: url-test                                    # 自动测延迟选最快
    url: http://www.gstatic.com/generate_204
    interval: 300                                     # 重测间隔（秒）
    tolerance: 50                                     # 容差，避免频繁抖动切换
    proxies: [香港01, 日本02, 新加坡03]
```

| 类型 | 行为 | 适用 |
| --- | --- | --- |
| `select` | 手动选，界面点哪个用哪个 | 需要固定出口时 |
| `url-test` | 自动测延迟选最快 | 日常使用 |
| `fallback` | 按顺序用，当前不可用才切下一个 | **已登录的账号类服务** |

> ⚠️ **`url-test` 会周期性切换出口 IP。** 如果你在用 AI 工具或其他已登录的账号服务，IP 频繁跳变容易触发风控甚至验证。这类流量应单独指向 `fallback` 或固定 `select` 分组。

### 链式代理（relay）

```yaml
proxy-groups:
  - name: 链式出口
    type: relay
    proxies:
      - 中转-香港      # 入口：线路质量好
      - 落地-美国原生   # 出口：IP 属性好
```

**它解决什么**：入口与出口分离——入口选线路好的中转，出口选 IP 属性好的落地。

**它不解决什么**：❌ 不等于匿名。若两跳属于同一家服务商，该服务商依然完整掌握「你是谁」和「你访问了什么」，多一跳没有切断关联。也不提升传输安全性（每跳本身已加密，叠加两层收益接近零）。

**代价**：延迟基本相加（60ms 链路做成链式常见到 150ms+）、故障点翻倍（可用性是两者乘积）、带宽取最慢一跳。

> 详解：[Clash 配置文件结构](https://clashvpnss.com/software_article_7.html) ·
> [链式代理的代价](https://clashvpnss.com/software_article_6.html)

---

## 三、分流规则实战

### 完整模板（注意顺序）

```yaml
rules:
  # 1. 局域网与本机，直连
  - IP-CIDR,192.168.0.0/16,DIRECT
  - IP-CIDR,127.0.0.0/8,DIRECT

  # 2. 明确要走代理的
  - DOMAIN-SUFFIX,google.com,PROXY
  - DOMAIN-SUFFIX,github.com,PROXY

  # 3. 明确要直连的（银行、支付等）
  - DOMAIN-SUFFIX,alipay.com,DIRECT

  # 4. 按 IP 归属兜底：国内直连
  - GEOIP,CN,DIRECT

  # 5. 剩余全部走代理
  - MATCH,PROXY
```

### ⚠️ 最常见的错误：规则顺序

`rules` **自上而下逐条匹配，命中第一条就停止**。这个「短路」特性是理解分流的关键。

> **典型错误**：把 `GEOIP,CN,DIRECT` 写在自定义规则**前面**。
> 后果：所有解析到国内 IP 的域名全部直连，你后面单独写的代理规则**永远不会生效**。

### 常用规则类型

| 类型 | 匹配对象 | 示例 |
| --- | --- | --- |
| `DOMAIN-SUFFIX` | 域名后缀 | `DOMAIN-SUFFIX,google.com,PROXY` |
| `DOMAIN-KEYWORD` | 域名含关键词 | `DOMAIN-KEYWORD,github,PROXY` |
| `DOMAIN` | 完整域名精确匹配 | `DOMAIN,www.google.com,PROXY` |
| `IP-CIDR` | IP 段 | `IP-CIDR,192.168.0.0/16,DIRECT` |
| `GEOIP` | IP 归属国家 | `GEOIP,CN,DIRECT` |
| `MATCH` | 兜底，**必须放最后** | `MATCH,PROXY` |

### DNS 配置（防污染）

```yaml
dns:
  enable: true
  enhanced-mode: fake-ip
  nameserver:
    - https://1.1.1.1/dns-query
    - https://dns.google/dns-query
```

用 DoH（DNS over HTTPS）让解析请求本身也走加密通道，绕过本地 DNS 污染。

> ⚠️ `fake-ip` 模式下，部分需要真实 IP 解析的场景（某些游戏、局域网设备发现）需加入 `fake-ip-filter` 白名单，否则会出现连接异常。

### 三个容易忽略的细节

1. **规则集比手写省心**：主流客户端支持引用社区维护的 `rule-provider`，涵盖常见站点分类，比逐条维护现实得多。
2. **对 IP 敏感的服务单独分组**：已登录账号频繁跳变出口 IP 容易触发风控，指向固定分组更稳。
3. **游戏/模拟器**：模拟器有自己的虚拟网络栈，宿主机系统代理不一定覆盖，通常需开 TUN 模式在网络层接管。

> 详解：[分流规则怎么配](https://clashvpnss.com/science_article_9.html) ·
> [进阶配置问答（62 条）](https://clashvpnss.com/faq-config.html)

---

## 四、客户端选型（分平台）

### 内核现状

原版 Clash 内核已于 **2023 年停止维护**，社区分支 **Clash Meta 接手并更名为 Mihomo**，兼容原配置格式，同时支持 VLESS、Hysteria 2、TUIC 等新协议。

> 选型时看一个简单指标：**项目最近一次提交是什么时候**。仍在用原版 Clash 内核的客户端，无法使用近两年出现的协议。

| 平台 | 推荐客户端 | 说明 |
| --- | --- | --- |
| **Windows / macOS / Linux** | Clash Verge Rev | 跨平台图形客户端，较活跃 |
| **Windows** | Mihomo Party | Mihomo 内核 |
| **macOS** | ClashX Meta | 轻量，Meta 内核 |
| **Android** | Clash Meta for Android | 社区维护主力 |
| **iOS** | Shadowrocket / Stash / Loon | 需 App Store 付费购买 |
| **路由器** | OpenWrt / iStoreOS + Mihomo | 全屋接管 |

> ⚠️ **旧的 Clash for Windows 已停止维护**，建议迁移。

### 下载安全（这一步风险最高）

**只从项目官方的 GitHub Releases 下载。** 代理客户端接管你的**全部网络流量**，权限极高，被植入后门的后果远超普通软件——而这类工具正是仿冒和捆绑的重灾区。

自查点：仓库地址与项目官方文档一致 · star 数与提交历史正常 · 下载后核对文件哈希。

> iOS 用户注意「**共享 Apple ID**」类购买方式：这意味着把账号交给他人，账号内其他数据同样面临风险，且随时可能被找回或锁定。

### 系统代理 vs TUN 模式

| | 系统代理 | TUN 模式 |
| --- | --- | --- |
| 原理 | 修改系统代理设置 | 创建虚拟网卡，网络层接管 |
| 覆盖范围 | 仅支持代理的程序（浏览器等） | **全部流量** |
| 局限 | 游戏、命令行工具、Docker 可能不走 | 需管理员权限，可能与 VPN/虚拟机网卡冲突 |

**怎么选**：只用浏览器，系统代理够了；发现某些程序不走代理（典型症状：浏览器能上但终端不行），再开 TUN。

> 详解：[Clash 生态现状](https://clashvpnss.com/software_article_9.html) ·
> [Shadowrocket 教程](https://clashvpnss.com/guide-shadowrocket.html) ·
> [安卓端教程](https://clashvpnss.com/guide-android.html) ·
> [Windows/macOS 教程](https://clashvpnss.com/guide-windows-mac.html)

---

## 五、故障排查决策树

### 「显示已连接，但打不开网页」

```
1. 先看订阅是否更新成功
   ├ 节点列表为空 / 还是旧的
   │  └→ 订阅链接过期，或订阅域名本身被墙
   │     （需先用其他方式连上一次再更新）
   └ 节点列表正常 → 下一步

2. 测试节点延迟
   ├ 全部超时 → 链路或账号问题
   └ 部分超时 → 换一个可用节点即可

3. 检查系统代理是否真的开启
   └ 有些客户端的开关与系统代理分开，
     软件显示已连接但系统并未走代理
     Windows: 设置 → 网络和 Internet → 代理

4. 确认系统时间准确  ← 最隐蔽但常见
   └ VMess 等协议对时间偏差敏感，
     偏差超过 90 秒直接握手失败
```

### 常见症状对照

| 症状 | 可能原因 | 处理 |
| --- | --- | --- |
| 节点全绿但网页打不开 | 系统代理未真正开启 / 系统时间偏差 | 校准时间、确认系统代理 |
| 微信能用，浏览器打不开国内网页 | Clash 劫持了 DNS 但解析失败 | 反复开关系统代理，或检查 DNS 配置 |
| 日志频繁 `connection refused` | 节点宕机，或订阅太久没更新导致端口已变 | 更新订阅 |
| 导入订阅提示 `invalid config` | 配置格式与客户端不匹配 | 确认订阅格式（Clash / sing-box / 通用） |
| 开杀毒软件后节点全 Timeout | 安全软件拦截 TUN 虚拟网卡或代理进程 | 将客户端加入白名单验证 |
| 关闭客户端后无法上网 | 客户端异常退出，系统代理设置未还原 | 手动关闭系统代理开关 |
| 能登录游戏但进不了对战 | 登录走 TCP 成功，实时同步走 UDP 失败 | 确认链路支持 UDP 转发 |
| 安卓锁屏后断连 | ROM 省电策略清理后台 | 设为「无限制」/「允许后台活动」 |
| macOS 系统代理开关自动弹开 | 新版系统网络权限收紧 | 系统设置 → 隐私与安全性 中授权 |

### 连不上的原因分类（不是所有问题都是配置）

| 现象 | 类型 | 应对 |
| --- | --- | --- |
| 域名解析到错误 IP，直连 IP 却通 | **DNS 污染** | 用 DoH / DoT |
| 特定 IP 完全不通，其他 IP 正常 | **IP 封锁** | 换节点即可 |
| 连接能建立，传输中途被中断 | **连接重置（RST）** | 流量特征被识别，需换协议而非换 IP |
| 新 IP 用几分钟又断 | 特征问题，非 IP 问题 | 换传输层方案 |

> 详解：[常见网络限制手段与应对](https://clashvpnss.com/science_article_7.html) ·
> [连不上与报错排查问答](https://clashvpnss.com/faq-troubleshooting.html)

---

## 六、自己验证的方法

比起相信宣传，这几件事你可以自己查。

### 1. 专线真伪（IPLC / IEPL）

```bash
# Linux / macOS，-r 报告模式，-c 发包次数
mtr -r -c 20 <节点IP>

# Windows：使用 WinMTR，或
tracert <节点IP>
```

**判断依据**：真正的专线从国内到境外**跳数很少**，且中间**不会出现公网骨干网 IP 段**。
若路由路径中出现 `202.97.x.x`（电信 163 骨干网）这类节点，说明流量走的是公网而非内网专线。

> ⚠️ 路由追踪结果可能受 ICMP 限制影响而不完整，个别跳显示星号是正常的。看整体路径特征，不要纠结单跳。

**成本逻辑也能反推**：IPLC/IEPL 是按带宽包月租用的固定资源，单价远高于公网流量。如果某套餐价格极低却宣称全专线，从成本上就难以成立——不需要懂技术，看定价逻辑就能发现问题。

### 2. CPU 是否支持 AES 硬件加速

决定你该选 `AES-256-GCM` 还是 `ChaCha20-Poly1305`：

```bash
# x86：输出中含 aes 即支持
grep -o 'aes' /proc/cpuinfo | head -1

# ARM：查看 Features 行是否含 aes
grep Features /proc/cpuinfo | head -1
```

**规则**：有 AES 硬件加速 → 用 AES-GCM；没有 → 用 ChaCha20。

几乎所有 2015 年后的桌面 CPU、主流 ARMv8 手机芯片都支持。真正需要 ChaCha20 的是部分低端路由器 SoC、嵌入式设备。**OpenWrt 软路由是最需要注意的场景**——不少型号 SoC 没有 AES 加速，选 AES 会让 CPU 成为吞吐瓶颈。

> 两者安全强度在实际使用中没有区别，都是经过充分审查的 AEAD 算法。**选择依据是硬件，不是安全性。**

### 3. 晚高峰表现

这是最关键也最容易被回避的一项。**务必自己在 20:00–24:00 测试**，白天的速度不说明任何问题——白天带宽富余，任何线路都跑得好看。

关注**延迟的稳定性**而非峰值速率：稳定的 150ms 体验尚可，而在 60ms 和 200ms 之间反复跳变的链路会让操作完全无法预判。

### 4. 游戏场景：NAT 类型

多人游戏尤其是 P2P 联机，NAT 类型决定能否成功连接：

| NAT 类型 | 说明 | 联机 |
| --- | --- | --- |
| **FullCone**（NAT1） | 任何外部主机都能主动连入 | 成功率最高 |
| Restricted / Port-Restricted | 只有先连过的对方能回连 | 部分场景可用 |
| **Symmetric**（NAT4） | 每个目标用不同端口映射 | P2P 打洞几乎必然失败 |

选节点时，**UDP 是否支持、NAT 类型是什么，比标称速率更值得关注**。

> 详解：[IPLC 与 IEPL 验证](https://clashvpnss.com/airport_article_4.html) ·
> [加密算法怎么选](https://clashvpnss.com/software_article_8.html) ·
> [游戏加速与 NAT](https://clashvpnss.com/guide-gaming.html)

---

## 七、术语速查

| 术语 | 含义 |
| --- | --- |
| **订阅链接** | 服务商给的一条 URL，客户端访问它自动获取全部节点。**等同于账号凭证，泄露即可被他人直接使用** |
| **节点** | 订阅里的每条线路，通常按地区命名 |
| **IPLC / IEPL** | 运营商提供的点对点专用电路，不经公网国际出口，不受晚高峰拥堵影响，成本高 |
| **公网中转** | 走运营商国际出口，成本低，晚高峰拥堵明显 |
| **原生 IP** | IP 注册归属地与服务器物理位置一致，且属当地正常住宅/商业网段，流媒体解锁能力好 |
| **落地节点** | 链式代理中最终出网的那一跳 |
| **流量倍率** | 消耗流量的计费系数，如 2 倍率表示用 1GB 扣 2GB |
| **AEAD** | 带认证的加密模式，在加密同时提供完整性校验（如 AES-256-GCM、ChaCha20-Poly1305） |
| **TUN 模式** | 创建虚拟网卡在网络层接管全部流量，不依赖程序是否支持代理 |
| **fake-ip** | DNS 返回虚假 IP 由代理内部映射，减少 DNS 泄漏并加快解析 |
| **DPI** | 深度包检测，分析数据包特征模式来识别代理流量 |
| **主动探测** | 审查方记下可疑地址后自己发起连接，根据服务端反应判定是否为代理 |
| **队头阻塞** | TCP 中前一个包丢失会阻塞后续包，UDP 系协议（QUIC）通过多路复用规避 |

> 更多：[基础概念问答](https://clashvpnss.com/faq-basics.html)

---

## 关于这份资料

- **不提供实测数据。** 这里没有延迟、带宽、丢包率的测试数值，也没有测速截图——那类结果依赖测试地点、时段、本地链路，别人测出来的数字对你没有参考意义。
- **给方法而不是结论。** 线路类型用 `mtr` 自验、硬件加速用 `/proc/cpuinfo` 自查、晚高峰自己在 20:00–24:00 测。
- **配置片段均可直接使用**，但请按自己的订阅和需求调整节点名称与域名列表。
- 技术在持续演进，**没有任何配置或协议是永久有效的**。发现错误或有补充，欢迎提 Issue 或 PR。

## 相关链接

- 完整教程与协议解析：[clashvpnss.com](https://clashvpnss.com/)
- 常见问题（200 条，按主题分册）：[clashvpnss.com/faq.html](https://clashvpnss.com/faq.html)
- 进阶配置问答（62 条）：[clashvpnss.com/faq-config.html](https://clashvpnss.com/faq-config.html)
- 连不上排查：[clashvpnss.com/faq-troubleshooting.html](https://clashvpnss.com/faq-troubleshooting.html)
- 机场选购方法（线路类型、计费、跑路风险）：[clashvpnss.com/airport.html](https://clashvpnss.com/airport.html)

## 利益披露

维护本仓库的 [clashvpnss.com](https://clashvpnss.com/) 通过推广链接获得佣金，
因此**并非无利益立场的第三方**——这也是本手册只讲配置与验证方法、
不提供任何「哪家更快」结论的原因。判断权在你。

站点的收入来源与立场说明：[关于我们](https://clashvpnss.com/about.html)

## 免责声明

本仓库为技术资料整理，不提供任何代理或网络服务，也不销售订阅。
内容仅供网络技术学习与研究，请遵守你所在国家和地区的法律法规。

## 许可

内容以 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 发布，转载请注明来源。
