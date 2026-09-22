# 数据源地图

实测时间：2026-09（亚运会男足赛事背景）。站点会变，遇到失效先按"技法"重试，再更新本文档。

## 一、可用源

### 1. 500.com —— 竞彩盘口

```bash
curl -s -m 20 -A "Mozilla/5.0 ..." "https://trade.500.com/jczq/" -o jczq.html
```

**编码坑**：页面声明 gb2312，但实际混入 UTF-8 片段，`utf-8`/`gb2312` 都会抛错。用 `gb18030` + `errors='replace'`（10 万字节里只有约 55 个替换符，不影响解析）。

**每场比赛是一个 `<tr class="bet-tb-tr" data-...>`**，关键属性：

| 属性 | 含义 |
|---|---|
| `data-matchnum` | 竞彩编号（如 `周二001`） |
| `data-matchtime` / `data-matchdate` | 开赛时间（**北京时间**） |
| `data-simpleleague` | 联赛/赛事简称 |
| `data-homesxname` / `data-awaysxname` | 主客队中文名 |
| `data-rangqiu` | **让球数**（如 `-1` 表示主队让 1 球） |
| `data-fixtureid` / `data-infomatchid` | 内部比赛 id |
| `data-isend` | 是否已结束 |
| `data-subactive` | 各玩法开售标记 |

`data-subactive` 示例与解读：

```
bfdg:1,bfgg:1,bqdg:1,bqgg:1,dxqdg:0,dxqgg:0,hcdg:1,hcgg:1,jqdg:1,jqgg:1,nspfdg:0,nspfgg:1,spfdg:0,spfgg:1
```

- `bf` 比分，`bq` 半全场，`dxq` **大小球**，`hc` 混合过关，`jq` 总进球数，`nspf` 胜平负单关，`spf` 胜平负过关
- `dg`/`gg` = 单关/过关；`0` 表示**该玩法未开售**

**所以"竞彩这期没开大小球"是可以直接从页面读出来的事实**，不是猜测。

**赔率取值**：每行有 6 个赔率格，顺序为 **胜平负(3) 在前、让球胜平负(3) 在后**。

读数的自我校验（务必做）：

```
1/(胜) + 1/(平) + 1/(负) ≈ 1.13     # 水位约 13%
1/(让胜) + 1/(让平) ≈ 1/(非让球胜)   # 若不等，说明盘口方向或字段顺序读错了
```

### 2. 雷速体育 —— 亚盘/大小球/欧赔平均值、情报、伤停、技术统计

**被阿里云 WAF 保护，`curl` 必失败，必须用浏览器**（`browser-use:control-browser`）。

curl 失败时的识别特征：返回约 20–24KB，内容含 `<textarea id="renderData">` 和 `acw_sc__v2` 字样。

#### 页面与路径

| 路径 | 内容 |
|---|---|
| `https://www.leisu.com/guide/swot-{id}` | **情报**（有利/不利/中立：伤停、阵容、动机、交锋要点）+ **盘口汇总** + 数据截止时间 |
| `https://live.leisu.com/detail-{id}` | 技术统计、事件流、实时比分 |
| `https://live.leisu.com/shujufenxi-{id}` | 历史交锋（含赛事类型）、近期战绩、联赛积分、进球分布、伤停情况、近期赛程、半全场胜负、动态（相关报道） |

#### matchId 从哪来

`https://www.leisu.com/` 首页的比赛列表**是服务器渲染的**（curl 可用），从中按中文队名/日期过滤即可拿到 `detail-{id}` 与 `swot-{id}`。列表文本形如：

```
亚运男足 小组赛 第3轮 未 韩国U23 沙特阿拉伯U23 今天 1...  → /detail-4594070
```

#### 盘口汇总的数据结构

页面内嵌数据里，`odds_list` 有三个子对象：

```js
odds_list.asia[0]   // [主水, 盘口, 客水]，例如 [1.02, 0.5, 0.77]
odds_list.asia[1]   // [主%, 客%]，进度条
odds_list.europe[0] // [主胜, 平, 客胜]，例如 [2.07, 2.9, 3.35]
odds_list.europe[1] // [主胜%, 客胜%]
odds_list.bs[0]     // [大球水, 进球盘口, 小球水]，例如 [1, 2.25, 0.8]
odds_list.bs[1]     // [大%, 小%]
```

**重要：只有一行即时值，没有初盘字段。** 所以"初盘 → 即时盘"无法从雷速回溯获取（见 SKILL.md 阶段 5：靠归档）。

盘口方向的校验：亚盘线为 `-0.5` 时主队必须赢才全赢，故主队水位应高于 1.00；若实测水位与该队欧赔隐含概率矛盾，说明读反了主客。

#### 技术统计（必须用 evaluate 读 DOM）

图表渲染，`domSnapshot()` 看不到数值。选择器：

```js
document.querySelectorAll(".bar-panel")   // 每行一项统计
  .left .tnum     // 主队数值
  .barcenter      // 统计名: 控球率 / 进攻 / 危险进攻 / 射门(射正) / 点球
  .right .tnum    // 客队数值

document.querySelector(".ts-top").querySelectorAll(".lab")  // 六个数，顺序固定:
  // 主队角球, 主队红牌, 主队黄牌, 客队黄牌, 客队红牌, 客队角球
```

#### 伤停模块

两种情况都要当有效结果记录：

- **「暂无伤停情况」** —— 这是权威源的正面答复，等价于"双方无伤停"，不是取数失败。
- 列出球员表：`球员 | 位置 | 原因 | 开始时间 | 归队时间 | 影响场数`，原因可能是"车祸""脚踝韧带撕裂"等；另有一张**停赛**表（红牌累计）。

伤停/停赛可以用技术统计里的**红牌数**交叉验证。

#### 相关报道

`shujufenxi` 与 `swot` 页面底部的**动态 / 相关资讯**会挂当天或前一日的媒体前瞻（如《足球报》分析对手阵容构成）。这类文章常含名单级细节（归化球员、国脚履历、停赛名单），价值很高，值得单独打开读。文章页是异步加载的，`waitForTimeout` 后需**重新 snapshot 一次**才能拿到正文。

#### 不要硬凑的点

- 「走势」等模块默认不加载，点击添加时 `.show-module` 有多个容器，`getByText` 容易匹配到 2 个元素而失败；元素还可能在视口外导致 click 超时。**不要反复重试同一个定位器**，这属于"拿不到"，如实标注。
- 若 `domSnapshot` 显示某模块标题但数值全空，先怀疑是图表/异步渲染，再试 evaluate。

### 3. Wikipedia —— 积分、赛果、大名单、射手榜、淘汰赛对阵

```
https://en.wikipedia.org/w/api.php?action=query&prop=extracts&explaintext=1&format=json&redirects=1&titles={条目}
```

**`prop=extracts` 会剥掉所有表格**，只剩正文段落。所以：正文用 API 取，**表格必须抓 HTML 用 BeautifulSoup/lxml 解析**。

```python
soup = BeautifulSoup(html, "lxml")
for s in soup(["sup", "style", "script"]):
    s.decompose()        # 必须: 否则脚注编号会粘进数字里
tables = soup.find_all("table")
h = tables[i].find_previous(["h2", "h3"])   # 用最近的标题定位是哪个小组/哪张表
```

- 小组积分表：`Pos | Team | Pld | W | D | L | GF | GA | GD | Pts`，队名后括号里 `(A)`/`(E)` 表示已晋级/已淘汰——**这是判断出线形势的现成字段**。
- 赛果：`div.footballbox`，含比分、进球者及分钟、半场、场地、观众、裁判。
- 大名单：通常在独立子页面（`… – Men's team squads`），主页面只有链接；子页面按小组+队名分节，表头 `No. | Pos. | Player | Date of birth | Club`，**`*` 标记超龄球员**。
- 淘汰赛对阵：bracket 里的 `C1 QF1 D2` 形式直接给出"小组名次 → 对手"的映射，是推算"争第一值不值"的关键。

### 4. TheSportsDB —— 赛程时间交叉验证

免费 key 为 `3`，放在路径里：

```bash
https://www.thesportsdb.com/api/v1/json/3/eventsday.php?d=2026-09-22&s=Soccer
https://www.thesportsdb.com/api/v1/json/3/eventsnext.php?id={teamId}
https://www.thesportsdb.com/api/v1/json/3/eventslast.php?id={teamId}
https://www.thesportsdb.com/api/v1/json/3/lookuptable.php?l={leagueId}&s={season}
https://www.thesportsdb.com/api/v1/json/3/searchteams.php?t={name}
```

- 时间戳是 **UTC**，适合做时区交叉验证。
- **覆盖不全**：小级别/综合运动会（如亚运会）只登记部分场次。不能作为唯一源，也不要用"这个源没有"推断"这场不存在"。

### 5. wttr.in —— 天气（仅赛前）

```bash
curl -s "https://wttr.in/Toyota?format=%l:+%C+%t+feels+%f+humidity+%h+wind+%w+precip+%p"
```

注意用**比赛所在城市**，不是常说的城市名（如决赛地在丰田市，不在名古屋）。

**`wttr.in` 只有实时/预报值，对已完赛场次无用。** 复盘任务要历史天气，用 Open-Meteo 归档接口：

```bash
# 经纬度 + 日期即可，返回 JSON，免费无需 key
curl -s "https://archive-api.open-meteo.com/v1/archive?latitude=35.08&longitude=137.15&start_date=2026-09-06&end_date=2026-09-06&daily=temperature_2m_max,temperature_2m_min,precipitation_sum,wind_speed_10m_max&timezone=Asia%2FTokyo"
```

### 6. Understat —— xG 与技术统计（五大联赛）

**必须加 `--compressed`**，否则拿到的是 gzip 二进制而不是 JSON。同时带浏览器 UA 与 `X-Requested-With`。

```bash
UA="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36"

# 单队整季：逐场结果 + xG/xGA
curl -s --compressed -A "$UA" -H "X-Requested-With: XMLHttpRequest" \
  "https://understat.com/getTeamData/Arsenal/2026"

# 整个联赛：所有球队的逐场历史
curl -s --compressed -A "$UA" -H "X-Requested-With: XMLHttpRequest" \
  "https://understat.com/getLeagueData/EPL/2026"
```

实测均返回 `200` + 真实 JSON。注意 `/main/getPlayersStats/` 是 **POST** 端点，GET 会返回 `{"error":{"error_code":4}}`。

联赛代号：`EPL` / `La_liga` / `Bundesliga` / `Serie_A` / `Ligue_1`。

**覆盖边界**：只有这五大联赛。**不覆盖**其他联赛、国家队赛事、综合运动会赛事——亚运会男足在这里查不到 xG，这是数据不存在，不是取数失败。

### 7. Transfermarkt —— 伤停、转会、阵容

`https://www.transfermarkt.com/` 实测 `200` 可达（带浏览器 UA）。球队页形如 `/{slug}/startseite/verein/{id}`。

用途：伤停与缺席球员、球队阵容与年龄结构、比赛页的首发与换人。**它没有亚运会级别的比赛页**——先用站内搜索确认覆盖，别猜 id（猜到的 id 会返回别的球队）。俱乐部赛事是它的强项。

### 8. Sky Sports / playerstats.football / 11v11 —— 俱乐部赛事战报与统计

已完赛的俱乐部赛事，这三个源能补齐雷速覆盖不到的部分：

- **Sky Sports 数据页**：控球、射门(射正)、禁区内外射门、角球、犯规、解围、扑救、xG/xGOT 与定位球拆分。战报页还含主帅与名宿引述。
- **playerstats.football**：球队技术统计逐项（可与 Sky 交叉）。
- **11v11**：交锋史（历史总战绩、纪录）与首发/换人细节。

注意源的字段可能出错（实测发现 Sky 的"抢断成功率 500%"这类明显异常值）——**异常值要剔除并在报告中注明**，不要照抄。

## 二、不可用源（不要浪费时间）

| 源 | 症状 | 结论 |
|---|---|---|
| `odds.500.com/fenxi/*` | 第一层 JS cookie（可算），第二层腾讯 EdgeOne **交互式验证码** | **不绕**，换源 |
| fbref.com | Cloudflare `403 "Just a moment..."` | 不可用 |
| worldfootball.net | 同上 | 不可用 |
| api.sofascore.com | `403 {"error":{"code":403}}` | 不可用 |
| fotmob.com `/api/*` | 全部 `404`（返回 Next.js 404 shell） | 不可用 |
| vip.titan007.com / www.nowgoal.com | `HTTP 000`（域名不可达） | 不可用 |
| okooo.com | 首页 `200` 但无盘口字段（JS 渲染） | 未找到比赛级盘口页 |
| data.7m.com.cn | 首页 `200`，只有联赛级 `odds_away{N}.shtml` 链接 | 未找到比赛级盘口页 |
| www.espn.co.uk | `202` 空响应体 | 不可用 |
| site.api.espn.com | `403` | 不可用 |
| Forza Football | `403` | 不可用 |

## 三、技法速查

| 症状 | 处理 |
|---|---|
| curl 拿到 ~20KB 空壳 + 混淆 JS | 阿里云 WAF，改用浏览器 |
| 页面显示 "Security Verification" + captcha 脚本 | 交互式验证码，**换源，不绕** |
| `domSnapshot()` 里字段名有、数值空 | 图表/异步渲染 → `playwright.evaluate` 读 DOM |
| 页面明显"加载中" | `waitForTimeout` 后**重新 snapshot**；SPA 内容不会推进旧快照 |
| click 报 `outside-viewport` 超时 | 先 `scrollIntoView` 或换用 evaluate；**不要重试同一 locator** |
| 定位器匹配到多个 | 用容器类名精确限定（如 `.show-module .module-name`），不要用 `first()`/`nth()` 硬凑 |
| Wikipedia 表格不见了 | `extracts` 剥表格，改抓 HTML |
| 中文页面解析报编码错 | 试 `gb18030` + `errors='replace'`；注意 gb2312 页面常混 UTF-8 |
| 拿到的是 gzip 二进制而非 JSON | curl 漏了 `--compressed`（Understat 等需要） |
| 报 `Browser is not available in subagent` | 浏览器不可用，走 SKILL.md 的"降级路径"，**不要跳过数据项也不要编** |
| 某源返回的比率明显荒谬（如 500%） | 源的字段错误，剔除并在报告中注明，不要照抄 |

## 四、更新本文档

源会失效、站点会改版。当你发现某个源的行为与本文档不符：**先按第 3 节排查**，确认是真实变更（不是姿势问题）后，更新对应条目，写清症状与新的解法。
