# 捡漏引擎 · Google Sheet 搭建说明（B）
**目标：建好 4 张表，让 `n8n_audio_picker_engine.json` 能跑起来**
版本 v1 · 2026-06-14

---

## 0. 总览
新建一个 Google 表格文件，名字随意（如 `Audio Picker DB`），在底部建 **4 个工作表标签**：

| 标签名 | 作用 | 谁写入 |
|---|---|---|
| `Watchlist` | 监控词表：引擎去搜什么 | 你手填 |
| `KB` | 需求知识库：引擎判断值不值得 | 导入 csv |
| `Deals` | 捡漏看板：命中的好货 + 视频脚本 | 引擎自动写 |
| `Seen` | 去重表：记录已处理过的链接 | 引擎自动写 |

> ⚠️ 标签名必须**完全一致**（区分大小写），n8n 节点里就是按这些名字找表的。
> ⚠️ 每张表的**第一行必须是表头**（列名），且列名要和下面**一字不差**（引擎按列名取值）。

---

## 1. 表 `Watchlist`（你手填）
引擎每轮读这张表，逐条去对应平台搜索。

**表头（A1 起，横向）：**
```
query_jp | platform | active
```

**字段说明：**
- `query_jp`：日文搜索词（品牌+配件+成色词组合）
- `platform`：`yahoo` 或 `hardoff`（先只用这两个）
- `active`：`TRUE` 启用 / `FALSE` 暂停

**示例行（可直接抄）：**
| query_jp | platform | active |
|---|---|---|
| サンスイ ジャンク | yahoo | TRUE |
| サンスイ ノブ | yahoo | TRUE |
| ナカミチ ヘッド | yahoo | TRUE |
| ナカミチ ピンチローラー | yahoo | TRUE |
| ラックスマン つまみ | yahoo | TRUE |
| マランツ ノブ | yahoo | TRUE |
| パイオニア SX ジャンク | yahoo | TRUE |
| DL-103 | yahoo | TRUE |
| ナガオカ 針 | yahoo | TRUE |
| 2SC3281 | yahoo | TRUE |
| 2SA1302 | yahoo | TRUE |
| ヤマハ NS-1000 ツイーター | yahoo | TRUE |
| ムギ球 メーターランプ | yahoo | TRUE |
| サンスイ ジャンク | hardoff | TRUE |
| ナカミチ ジャンク | hardoff | TRUE |

> 起步 10~15 条够了。跑顺后再扩。

---

## 2. 表 `KB`（导入 csv）
这是引擎的"筛选大脑"，**别手敲**，直接导入：

**导入方法（不乱码）：**
1. 打开 Google 表格 → 选中 `KB` 标签
2. 菜单 **文件 → 导入 → 上传** → 选 `audio_demand_kb.csv`
3. 导入位置选 **"替换当前工作表"**
4. 分隔符选 **"逗号"**
5. 确定

> Google Sheet 读 UTF-8 正常，中文不会乱码（本地 Excel 才会，那是另一回事）。

**这张表的列（导入后自动就有，供你核对）：**
```
brand | jp_name | category | hot_models | demand_tier |
resale_usd_low | resale_usd_high | target_max_buy_jpy |
ship_class | voltage_relevant | fake_risk | notes
```
- `demand_tier`：S/A/B（需求等级，影响打分权重）
- `resale_usd_low/high`：欧美售价区间（美元）← **后续用真实 eBay sold 校准**
- `target_max_buy_jpy`：目标进货价上限（超过就别抢）
- `ship_class`：light/med/heavy（运输等级，影响运费和风险扣分）

> 以后想加品牌/品类：**在这张表加一行就行**，引擎下轮自动按它筛。

---

## 3. 表 `Deals`（引擎自动写，你只建表头）
命中的好货落这里，含自动生成的视频脚本。**你只需把表头建好**，数据引擎来填。

**表头（A1 起）：**
```
title | buy_price_jpy | url | image | brand | category | model |
demand_tier | profit_jpy | margin_pct | deal_score | hot |
harvest_parts | selling_points | video_script | created_at
```

> 列名要和引擎输出的字段对得上（上面这些就是引擎算出来的字段）。
> `video_script` 列会放 LLM 生成的脚本；`created_at` 可在 n8n 里加个时间戳节点写入。

---

## 4. 表 `Seen`（引擎自动写，你只建表头）
防止同一个商品被重复预警。

**表头（A1 起）：**
```
url | seen_at
```

**去重逻辑（在 n8n 里加，写入 Deals 前）：**
- 引擎拿到一条商品的 `url`
- 先查 `Seen` 表里有没有这个 `url`
- 有 → 跳过；没有 → 处理 + 把 `url` 写进 `Seen`

> n8n 实现：用一个 "Google Sheets - Lookup" 节点查 url，接 IF 节点判断是否已存在。
> 这一段我可以在工作流 JSON 里帮你补上，你说一声。

---

## 5. 拿到 Sheet ID（填进 n8n）
浏览器打开你这个 Google 表格，地址栏长这样：
```
https://docs.google.com/spreadsheets/d/【这一段就是 SHEET_ID】/edit#gid=0
```
把中间那段复制出来，替换 `n8n_audio_picker_engine.json` 里所有的 `YOUR_SHEET_ID`。

---

## 6. n8n 凭据清单（要配 3 个）
| 凭据 | 用途 | 在哪配 |
|---|---|---|
| Google Sheets OAuth2 | 读写 4 张表 | n8n Credentials → Google Sheets |
| Anthropic API Key | 生成视频脚本 | n8n Credentials → Anthropic |
| Telegram Bot Token | 推手机预警 | 找 @BotFather 建 bot 拿 token + chatId |

---

## 7. 第一次跑（验证顺序，别一次全开）
1. **只留 `Watchlist` 里 1 条 yahoo 词**（如 `DL-103`），其余设 `active=FALSE`
2. n8n 里**手动执行**工作流一次
3. 看 `抓取Yahoo拍卖` → `解析商品列表` 节点有没有抓到数据
   - 抓不到 → 多半是 CSS 选择器要调（Yahoo 改版），把节点输出截图给我，我帮你改
4. 看 `评分引擎` 输出的分数对不对
5. OK 后再逐步打开更多词 + 加 Hard-Off + 接脚本生成 + 接 Telegram

---

## 8. 常见坑
- ❌ 标签名/列名写错 → 引擎取不到值。**逐字核对**。
- ❌ 一上来开几十个词、间隔太短 → 可能被平台限频/封 IP。**起步 15 词、20 分钟一轮、加随机延迟**。
- ❌ KB 的 `resale_usd` 还是示意值 → 判断会偏。**尽快用真实截图校准（步骤 A）**。
- ❌ Mercari 先别接 → 反爬重，二期再说。

---

## 下一步
表建好后，进入 **A：校准知识库**——你抓一批真实 Yahoo 在售价 + eBay 已售价截图给我，我把 `resale_usd` 和 `target_max_buy_jpy` 改成真数字，引擎判断就准了。
