---
name: moontv-daily
description: MoonTV 每日影视精选 — 抓取 24h 最新影视数据，按电影/剧集/综艺/短剧/福利五类各精选 5 部，支持追剧列表自动更新检测，输出带评分和播放链接的推荐报告。
user-invocable: true
allowed-tools:
  - "WebFetch(*)"
  - "Bash(*)"
  - "Write(*)"
  - "Read(*)"
  - "Glob(*)"
---

# MoonTV 每日精选 (MoonTV Daily Elite Feed)

## 概述

本技能从 MoonTV 网关获取全部可用 CMS 源站，并发抓取 24 小时内更新的影视数据，按 `vod_name` 去重合并后，通过关键词分类、质量过滤、双路径加权评分排序，输出"电影/剧集/综艺/短剧/福利"五类各 Top 5 的精选推荐报告。

**核心价值**：
- 全量聚合：并发抓取所有 CMS 源站，按片名去重，消除单源遗漏
- 极简推送：每类仅 5 部，共 25 部黄金片源
- 智能排序：豆瓣评分 + 热度 + 时效性双路径加权算法
- 双路由交互：播放页直达链接，剧集支持最新集秒播

## 触发方式

- **手动**：用户输入 `/moontv` 即时触发
- **定时**：使用 CronCreate 设置每日定时推送（见末尾模板）

## 环境变量

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `MOONTV_GATEWAY` | 网关地址 | `https://moontv-api.12879737.xyz` |
| `MOONTV_PLAY_URL` | 播放页地址 | `https://moontv.dduan2002cn.xyz/play` |
| `WATCHLIST_FILE` | 追剧配置文件路径 | `config/watchlist.json` |

---

## 执行步骤

当用户触发本技能时，严格按以下步骤执行：

### 步骤 1：获取源站列表

使用 WebFetch（或 Bash curl）访问网关获取可用资源站列表：

```
URL: ${MOONTV_GATEWAY}
```

返回的 JSON 中 `api_sites` 数组包含所有可用 CMS 源站（按探活速度排序）。

**错误处理**：
- 如果网关不可达 → 输出错误信息并终止
- 如果返回空数组 → 输出"无可用资源站"并终止

### 步骤 2：全量聚合 — 并发抓取所有源站

**并发**访问所有源站的 API（使用 Bash + Python 并发，或逐个 WebFetch）：

```
URL: ${site.api}?ac=detail&h=24&pg={page}
```

每个源站需翻页抓取全量数据（`pagecount` 字段指示总页数）。

**关键字段**：
- `vod_id`：视频 ID（用于构建播放链接）
- `vod_name`：片名（用于去重）
- `type_name`：分类名（如"动作片"、"国产剧"、"大陆综艺"）
- `vod_remarks`：清晰度/更新状态（如"HD 中字"、"更新至第10集"）
- `vod_douban_score`：豆瓣评分（0 表示无评分）
- `vod_hits`：播放热度
- `vod_time`：入库时间（时间戳或日期字符串）
- `vod_year`：年份（用于构建播放链接）

**去重**：按 `vod_name` 去重，保留首次出现的记录。

**错误处理**：
- 单个源站失败 → 跳过该源，继续其他源站
- 所有源站都失败 → 输出错误信息并终止
- 去重后无数据 → 输出"今日无更新数据"并终止

### 步骤 2.5：追剧匹配

从 `config/watchlist.json` 读取用户追剧列表，在已抓取的全量数据中匹配更新。

#### 配置文件格式

```json
{
  "watchlist": [
    {"name": "大唐迷雾", "type": "剧集"},
    {"name": "认识的哥哥", "type": "综艺"}
  ]
}
```

- `name`：剧集/综艺名称关键词（模糊匹配 `vod_name`）
- `type`：可选，辅助过滤（不参与匹配逻辑）

#### 匹配流程

**路径 A — 从已抓取数据中匹配（零成本）**：

```python
def match_watchlist(watchlist, all_items):
    results = []
    found_names = set()
    for w in watchlist:
        keyword = w["name"]
        for item in all_items:
            if keyword in item.get("vod_name", ""):
                results.append({"watch": w, "item": item})
                found_names.add(keyword)
                break
    missing = [w for w in watchlist if w["name"] not in found_names]
    return results, missing
```

**路径 B — 搜索 API 兜底（对路径 A 未命中项）**：

对 `missing` 中的每个剧目，并发搜索所有源站：

```
URL: ${site.api}?ac=detail&wd={encodeURIComponent(name)}
```

取第一个返回有效结果的源站数据。

#### 输出要求

- 追剧结果**不参与评分排序**，单独展示在报告顶部
- 每部追剧显示：片名、最新更新状态、豆瓣评分、播放链接
- 标注"有更新"vs"无更新"
- 如果 `watchlist.json` 不存在或为空，跳过此步骤，不影响后续流程

### 步骤 3：数据处理

对去重后的数据执行以下处理：

#### 3.1 分类

根据 `type_name` 关键词匹配分为五类：

| 优先级 | 目标分类 | 匹配规则 |
|-------|---------|---------|
| 1 | 🎁 福利专区 | `type_name` 包含 `福利` |
| 2 | 🎭 热门短剧 | `type_name` 包含 `短剧` |
| 3 | 🎞️ 热门电影 | `type_name` 包含 `片` 或 `电影` |
| 4 | 📺 热门剧集 | `type_name` 包含 `剧` |
| 5 | 🍲 热门综艺 | `type_name` 包含 `综艺` 或 `真人秀` |

**匹配优先级**：按上表从上到下顺序匹配，先匹配到的优先归类。关键：`短剧` 必须在 `剧` 之前检查，否则"短剧"会被误归入"剧集"。
**不匹配任何规则的数据直接丢弃**（如动漫、纪录片、足球等）。

#### 3.2 过滤

在每个分类中，剔除以下低质内容：

从 `vod_remarks` 中剔除包含以下关键词的记录：
- `TC`、`枪版`、`预告片`、`广告`

从 `type_name` 中剔除包含以下关键词的记录：
- `AI漫剧`、`解说`

#### 3.3 双路径评分

对每个分类内的数据，按以下公式计算综合推荐指数：

**权重常量（可人工调整）**：
```
# 有评分路径
W_DOUBAN = 0.6
W_HOT = 0.3
W_RECENCY = 0.1

# 无评分路径
W_HOT_NO_SCORE = 0.6
W_RECENCY_NO_SCORE = 0.3
W_BASE_NO_SCORE = 0.1

# 常量
BASE_SCORE = 5.0
DEFAULT_DOUBAN = 5.5
RECENCY_WINDOW_HOURS = 12
TOP_N = 5
```

**有豆瓣评分**（`vod_douban_score > 0`）：
```
Score = (vod_douban_score × 0.6) + (归一化热度 × 0.3) + (时效 × 0.1)
```

**无豆瓣评分**（`vod_douban_score == 0`）：
```
Score = (归一化热度 × 0.6) + (时效 × 0.3) + (BASE_SCORE × 0.1)
```

**各维度计算**：
- 豆瓣评分：直接使用 `vod_douban_score`
- 归一化热度：`(vod_hits / 分类内最大 vod_hits) × 10`
- 时效红利：`vod_time` 在最近 12 小时内 → 10，否则 → 0

#### 3.4 排序截取

每个分类按 Score 降序排列，取前 5 条（`.slice(0, 5)`）。

### 步骤 4：格式化输出

#### 4.1 清理历史文件

在写入新文件前：
1. 使用 Glob 扫描 `output/moontv-daily-*.md`
2. 解析文件名中的日期（YYYY-MM-DD）
3. 删除 7 天前的旧文件

#### 4.2 生成报告

按以下格式生成 Markdown 报告，同时输出到终端和文件：

```markdown
# 🎬 MoonTV 每日精选 [YYYY-MM-DD]

> 数据源：全量聚合（{N} 站）| 原始 {X} 条 → 去重 {Y} 条 → 精选 {Z} 部
>
> 各站数据：量子资源(175) | 豪华资源(171) | ...

---

## 📋 追剧更新（{M}/{N} 有更新）

| 剧名 | 最新进度 | 豆瓣 | 播放 |
|------|---------|------|------|
| {片名} | {更新状态} | ⭐{评分或 -} | [▶️ 播第X集](播放链接) |

> 未更新：{未更新的剧名列表}

（如果 watchlist.json 不存在或为空，此板块不显示）

---

## 🎞️ 热门电影（Top 5）

1. **《{片名}》** {清晰度} ⭐{评分}
   💡 {一句话亮点：基于片名和简介生成，突出看点}
   🔗 [▶️ 播放](${MOONTV_PLAY_URL}?id={vod_id}&title=${encodeURIComponent(片名)}&year={年份}&stype=movie)

（重复 5 条，或显示"今日暂无更新"如果不足 5 条）

---

## 📺 热门剧集（Top 5）

1. **《{片名}》** {更新状态} ⭐{评分}
   💡 {一句话亮点}
   🔗 [▶️ 播第{X}集](${MOONTV_PLAY_URL}?id={vod_id}&title=${encodeURIComponent(片名)}&year={年份}&stype=tv&ep={X})

**播放链接规则**：
- 电影 → `stype=movie`，显示 `[▶️ 播放]`
- 连载中剧集/综艺/短剧 → `stype=tv`，显示 `[▶️ 播第X集]`（附 `&ep={X}`）
- 已完结 → `stype=tv`，仅显示 `[▶️ 播放]`（不附 ep 参数）
- 集数提取：优先匹配 `第(\d+)集`，忽略日期格式（如 `20260531期`）

---

## 🍲 热门综艺（Top 5）

（格式同剧集，同样支持双路由）

---

## 🎭 热门短剧（Top 5）

（格式同剧集，支持双路由）

---

## 🎁 福利专区（Top 5）

（格式同剧集，支持双路由）

---

📊 聚合简报：{N} 站并发 → 原始 {X} 条 → 去重 {Y} 条 → 过滤 {M} 条 → 精选 {Z} 部
```

#### 4.3 写入文件

将报告写入 `output/moontv-daily-YYYY-MM-DD.md`（使用 Write 工具）。

#### 4.4 终端展示

将同样的 Markdown 内容直接输出到终端供用户阅读。

---

## 亮点生成规则

为每部影视生成一句话亮点时，遵循以下原则：
- 基于 `vod_name` 和剧情简介（如有）生成
- 突出看点：知名演员、导演、奖项、题材特色
- 控制在 30 字以内
- 不要编造虚假信息，如无足够信息则写"精彩内容，不容错过"

---

## 定时推送设置

用户可选择启用每日定时推送。在 Claude Code 中执行以下命令：

```
CronCreate:
  cron: "0 9 * * *"
  prompt: "/moontv"
  recurring: true
  durable: true
```

这将在每天上午 9:00 自动执行本技能，推荐结果持久化保存。

---

## 错误处理

| 场景 | 处理方式 |
|------|---------|
| 网关不可达 | 输出错误信息，提示检查 Worker 状态，不写文件 |
| 网关返回空列表 | 输出"无可用资源站"，建议稍后重试 |
| CMS API 返回异常/超时 | 输出错误详情，不生成推荐 |
| 某分类数据为 0 条 | 该分类显示"今日暂无更新"，不影响其他分类 |
| 所有分类都为空 | 输出"今日无有效更新数据" |
| WebFetch 域名校验失败 | 提示用 Bash(curl) 作为备选方案 |
| watchlist.json 不存在 | 跳过追剧匹配，正常执行后续步骤 |
| watchlist.json 格式错误 | 输出警告，跳过追剧匹配 |
| 追剧搜索全部无结果 | 显示"未找到匹配内容"，不影响主流程 |

**关键原则**：任何步骤失败都不写入文件、不清理旧文件，确保不会产出空报告或损坏报告。

---

## 备选方案：Bash(curl)

如果 WebFetch 因域名校验失败无法访问 API，可使用 Bash 作为备选：

```bash
# 步骤 1：获取最快资源站
curl -s "https://moontv-api.12879737.xyz"

# 步骤 2：抓取数据（替换 API 地址）
curl -s "https://cj.lziapi.com/api.php/provide/vod/?ac=detail&h=24"
```

---

## 迭代记录

| 版本 | 日期 | 变更 |
|------|------|------|
| v1.0 | 2026-05-31 | 初始版本：三分类各 Top 5，双路径评分，双路由链接 |
| v1.1 | 2026-06-01 | 新增短剧/福利分类（共五类），移除 source 参数，优化集数提取正则 |
| v1.2 | 2026-06-01 | 全量聚合：并发抓取所有源站，按 vod_name 去重合并，数据量从 180→393 |
| v1.3 | 2026-05-31 | 追剧功能：支持 config/watchlist.json 配置追剧列表，自动匹配更新并展示在报告顶部 |
