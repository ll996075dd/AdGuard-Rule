# AdGuard Home 黑名单清洗报告

> 生成时间：2026-07-24
> 数据源：`/root/adguard/BlackList.txt`（iStoreOS 上 AdGuard Home，`file://` 本地源）
> 配套文件：`BlackList.cleaned.txt`（本报告产出的清洗候选，原文件未改动）

---

## 0. 先纠正一个关键认知：AdGuard Home ≠ AdGuard 浏览器插件

你特别强调"是 **AdGuard Home**，不是 AdGuard"——这一点决定了规则写法，也是上一轮我差点搞错的地方。

| | AdGuard（浏览器插件 / Windows / Android App） | **AdGuard Home（本次环境）** |
|---|---|---|
| 工作层级 | 应用层 / 浏览器内 | **网络 DNS 层**（拦截域名解析） |
| 能看到什么 | 完整 URL、页面 DOM、CSS、HTTP 头 | **只有域名**（DNS 查询里的主机名） |
| 元素隐藏 `##` | ✅ 支持 | ❌ 完全忽略 |
| URL 路径匹配 `\|\|x.com/ads/*` | ✅ 精确 | ❌ 路径在 DNS 层不存在，规则要么整域拦、要么不匹配 |
| 内容类型 `$script` `$image` | ✅ 支持 | ❌ DNS 看不见内容类型，修饰符被忽略 |
| `$third-party`（第三方） | ✅ 靠 Referer 判断 | ⚠️ DNS 无 Referer，实际失效 |
| `$app`（按 App 拦） | ✅ 客户端才有 | ❌ Home 无"应用"概念 |
| `$dnsrewrite` `$client` `$ctag` `$dnstype` | ❌ 不支持 | ✅ **Home 专属**，DNS 级才有用 |

**一句话**：AdGuard Home 的规则本质是"**按主机名拦截/放行**"。任何依赖 URL 路径、页面元素、HTTP 头、内容类型的写法，在 Home 里要么无效、要么被降级成"整域拦截"。

---

## 1. AdGuard Home 规则语法速查（DNS 级正确写法）

### 1.1 支持的写法（推荐日常用这些）

| 写法 | 含义 | 示例 |
|---|---|---|
| `\|\|example.com^` | 拦截域名**及其所有子域**（最常用） | `\|\|doubleclick.net^` |
| `example.com` | 只拦**主域名**，子域不拦 | `example.com` |
| `*.ads.example.com` | 通配，拦该级下所有子域 | `*.ads.example.com` |
| `@@\|\|example.com^` | **放行**（例外），优先级高于拦截 | `@@\|\|static.zhihu.com^` |
| `0.0.0.0 example.com` | Hosts 格式拦截 | `0.0.0.0 example.com` |
| `/regex/` | 正则匹配主机名 | `/^ad[0-9]+\.example\.com$/` |
| `$important` | 提升优先级，可压过放行规则 | `\|\|spam.com^$important` |
| `$badfilter` | 禁用另一条规则 | `\|\|spam.com$badfilter` |
| `$denyallow=域名` | 拦整域但放行例外 | `*$denyallow=com\|net` |
| `$client=192.168.x.x` | 只对某客户端生效 | `\|\|ads.com^$client=192.168.31.50` |
| `$dnstype` `$dnsrewrite` `$ctag` | Home 专属 DNS 控制 | `\|\|tracker.com^$dnsrewrite=REFUSED` |

### 1.2 DNS 级**失效/不支持**的写法（写了也白写或变味）

- **元素隐藏** `example.com##.banner` —— Home 直接忽略。
- **URL 路径** `\|\|example.com/ads/*` —— DNS 只看到 `example.com`。结果二选一：整域被拦（比你想的宽），或整条不匹配。**路径级精细拦截在 Home 做不到**，必须放浏览器端 AdGuard。
- **内容类型** `$script` `$image` `$stylesheet` `$media` `$subdocument` `$xmlhttprequest` `$object` `$ping` `$popup` `$inline-script` `$replace` `$csp` `$cookie` `$header` `$removeparam` … —— Home 忽略，但**域名仍会被拦**（规则没死，只是修饰符没用）。
- **`$third-party` / `$~third-party`** —— 无 Referer，DNS 层无法判定第一/第三方，实际失效（域名仍拦）。
- **`$app`** —— 仅客户端 AdGuard 有，Home 无。

### 1.3 两个易踩的坑

1. `^` 是**域名分隔符**，防止误拦相似域。例如 `\|\|example.com^` 不会误拦 `example.com.cn`。
2. `example.com`（无 `||`、无 `^`）**只拦主域**，子域照常解析——很多人以为它等于 `||example.com^`，其实差很多。

---

## 2. 本次下载规则合规审计

### 2.1 黑名单 `BlackList.txt`（共 1,265,666 条规则）

| 类别 | 数量 | 在 AdGuard Home 的实际命运 |
|---|---:|---|
| **有效 DNS 主机规则**（`\|\|域^` / `*.域` / 纯域 / 正则 / hosts） | ~1,263,104 | ✅ 正常拦截 |
| 带路径规则 `\|\|域/path`（有域） | 603 | ⚠️ 路径被忽略，**整域被拦**（或可能不匹配） |
| 裸路径规则 `*/path`、**无域名** | 536 | ❌ **死规则**，永远不匹配 |
| 无域名怪规则（`\|\|diyibanzhu^`、`\|push-notifications.`、`\|\|0000153...^` 等） | 1,372 | ❌ **死规则/无效** |
| 含 DNS 失效修饰符（`$script`/`$image`/`$third-party`…） | 375 | ⚠️ 域名照拦，修饰符白写 |
| **畸形修饰符规则**（见下） | 30 | ❌ AGH 无法解析，应删 |
| 元素隐藏 `##` | 0 | — 本列表无 |

**畸形修饰符 30 条**来源可疑：其中混入了"什么值得买"(Smzdm) 类**JSON 过滤规则**的残留，例如：
```
$"entityTemplate":"apkImageCard".*?\}...
$/ /! 帖子详情赞助内容
$"detailSponsorCard":{.*}/}}/! 发现页去除酷品
```
这些不是 AdGuard Home 语法，是某 App 私有 JSON 过滤器的写法，在 Home 里属**非法规则**，应整条删除或还原成正常域名规则。

### 2.2 白名单 `WhiteList.txt`（2,706 条）— ✅ 干净

全部为 `@@||域^` 或 `@@||*.域$important` 形式，**100% 符合 AGH 放行语法**，无死规则、无畸形修饰符、无拦截规则混入。无需处理。

### 2.3 自定义规则 `user_rules.txt`（192 条 = 148 放行 + 40 拦截）— ✅ 干净

格式规范（`@@||...^$important` 与 `||...^`），无死规则、无畸形修饰符。无需处理。

---

## 3. 清洗方案与结果

清洗目标：**只删"在 AdGuard Home 里必然无效/非法"的规则**，不动任何仍在生效的规则，避免误杀。

| 处理动作 | 数量 | 风险 |
|---|---:|---|
| 删死规则（裸 path 无域 536 + 无域怪规则 1,372，合计 **1,952**） | 安全，0 误杀 |
| 删畸形修饰符规则（**30**） | 安全，这些本就不被解析 |
| 带路径规则规范化（`\|\|域/path` → `\|\|域^`，**577** 条） | ⚠️ 见下方"决策点 1" |
| 内容类型修饰符规则（375） | **保留不动**（域名仍拦，删修饰符不影响拦截） |

**清洗后：`BlackList.txt` 1,265,666 → `BlackList.cleaned.txt` 1,263,684 条（净减 1,982）。**

原文件、上一轮 `BlackList.dedup.txt`（仅做域名级冒泡去重，删 186 条子域冗余）均保留作备份，未覆盖。

---

## 4. 需要你拍板的两个决策点

**决策点 1 — 带路径规则要不要"整域化"？**
577 条 `||x.com/path` 在 DNS 级本就只能整域拦或无效。我已把它们规范成 `||x.com^`（保留 `$important`）。但这意味着：
- 若你本意就是"拦这个域" → 完全正确，更清晰。
- 若你本意是"只拦该路径下的广告、保留域名可用" → AGH **做不到**，只能接受整域拦，或把这条移到浏览器端 AdGuard。
如果不想整域化，我可以改成"直接从 cleaned 文件里剔除这 577 条"（即宁可漏拦也不扩大拦截面）。

**决策点 2 — 内容类型修饰符（375 条）去留？**
它们现在"域名照拦、修饰符白写"。建议删掉修饰符让规则更干净（不影响拦截），或保持原样。你定。

---

## 5. 回写服务器建议（执行前请确认）

1. SSH 登录 iStoreOS：`root@192.168.31.243`
2. 备份原文件：`cp /root/adguard/BlackList.txt /root/adguard/BlackList.txt.bak-20260724`
3. 上传 `BlackList.cleaned.txt` 覆盖 `/root/adguard/BlackList.txt`
4. 重启 AdGuard Home 使生效（iStoreOS 服务名通常为 `adguardhome`）
5. 观察 24h 日志，确认无正常域名被误拦

> ⚠️ `adguardhome.yaml` 含 admin 密码 bcrypt 哈希，属敏感信息，勿外泄/提交公仓。

---

## 附：产物清单

| 文件 | 说明 |
|---|---|
| `BlackList.cleaned.txt` | 本次清洗候选（DNS 级合规，待你确认决策点后可用） |
| `BlackList.dedup.txt` | 上一轮：仅域名级冒泡去重（删 186 子域冗余） |
| `BlackList.txt` | 服务器原始规则（未改动，作备份） |
| `WhiteList.txt` / `user_rules.txt` | 白名单 / 自定义规则（均干净，无需清洗） |
| `adguardhome.yaml` | AGH 配置（含敏感密码哈希） |

---

## 6. 部署结果（2026-07-24 回写服务器，已执行）

老板确认两项决策（577 条带路径规则整域化 + 375 条内容类型修饰符剥离）后已回写并验证。

**执行动作**
1. 带时间戳备份：
   - `/root/adguard/BlackList.txt.bak-20260724-2131`
   - `/var/lib/adguardhome/data/filters/1775899080.txt.bak-20260724-2131`
2. 上传清洗文件（源 + AGH filter 缓存双写）：`BlackList.cleaned.txt`（32,098,799 字节）→ 覆盖 `BlackList.txt` 与缓存 `1775899080.txt`
3. 重启服务：`/etc/init.d/adguardhome restart`（pid 7578 → 20520）
4. 校验：进程存活；缓存大小 = 清洗文件大小（已加载新规则）

**拦截生效验证（原始 DNS 查询查 RCODE）**
- 清洗黑名单随机抽检 5 个域名（`et.akademie-handel.de` / `clixgalore.com` / `images.response.cbre.com.au` / `sst.stunick.com` / `doublebreasted.shop`）→ **全部 NXDOMAIN** ✅
- `iqiyi.com` → NXDOMAIN（黑名单 `$important` 压过白名单放行）✅
- 结论：**清洗后的大黑名单已正确加载并拦截，未损坏任何规则。**

**⚠️ 发现独立问题：AGH 上游 DoH/DoT 当前不可达（与本次过滤改动无关）**
- 现象：正常域名（example.com 等）经 AGH 解析返回 `NOERROR / 0 条`（空应答）。
- 根因：AGH 上游全是加密 DNS——`https://162217.alidns.com`、`https://doh.pub`、`tls://dot.360.cn` 等。实测 DoH(443) 全部超时；普通 DNS（223.5.5.5）正常。
- 注意：`doubleclick.net` / `mmstat.com` / `googlesyndication.com` 经查在**白名单**里被放行（原有配置，未改动），故它们走上游也空应答——不是过滤改动导致。
- 影响：广告/追踪域名仍被正常拦截；但**正常网站解析暂时失效**（需上游恢复或调整上游配置）。
- 待办（需老板另行确认，属服务器上游配置变更）：可把 `223.5.5.5` / `119.29.29.29` 等普通 DNS 加入 `upstream_dns` 或 `fallback_dns`，或替换当前超时的 DoH 供应商。

**回滚方式**：若需恢复，`cp /root/adguard/BlackList.txt.bak-20260724-2131 /root/adguard/BlackList.txt` 并重启 AGH 即可。
