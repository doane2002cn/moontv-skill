# 🎬 MoonTV Daily Elite Feed

> **MoonTV 每日影视精选** — A Claude Code Skill for daily curated movie & TV recommendations

[![Claude Code](https://img.shields.io/badge/Claude-Code-blue)](https://claude.ai/code)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## 📖 项目简介 | Introduction

### 中文

MoonTV 每日精选是一个 Claude Code 技能，能够自动从多个 CMS 影视资源站抓取最新数据，通过智能去重、分类、评分算法，每日精选出 25 部优质影视内容（电影/剧集/综艺/短剧/福利各 Top 5）。

**核心价值**：
- 🔄 **全量聚合**：并发抓取所有 CMS 源站，按片名去重，消除单源遗漏
- ⭐ **智能排序**：豆瓣评分 + 热度 + 时效性双路径加权算法
- 📺 **追剧追踪**：支持配置追剧列表，自动检测更新
- 🔗 **直达播放**：生成播放页链接，剧集支持最新集秒播

### English

MoonTV Daily Elite Feed is a Claude Code skill that automatically fetches the latest data from multiple CMS movie/TV resource sites. Through intelligent deduplication, categorization, and scoring algorithms, it curates 25 premium content items daily (Top 5 for each category: Movies, TV Shows, Variety Shows, Short Dramas, and Adult Content).

**Core Values**:
- 🔄 **Full Aggregation**: Concurrently fetches all CMS sources, deduplicates by title, eliminates single-source gaps
- ⭐ **Smart Sorting**: Dual-path weighted algorithm combining Douban ratings, popularity, and recency
- 📺 **Watchlist Tracking**: Configure watchlist to automatically detect updates for your favorite shows
- 🔗 **Direct Playback**: Generates play links with support for latest episode quick-play

---

## ✨ 功能特性 | Features

| Feature | Description |
|---------|-------------|
| 🌐 多源聚合 | 并发抓取所有可用 CMS 源站，数据量从单源 180+ 提升至全量 400+ |
| 🔍 智能去重 | 按 `vod_name` 去重，保留首次出现的记录 |
| 📊 五分类推荐 | 电影、剧集、综艺、短剧、福利各 Top 5 |
| ⭐ 双路径评分 | 有豆瓣评分：评分×0.6 + 热度×0.3 + 时效×0.1；无评分：热度×0.6 + 时效×0.3 + 基准×0.1 |
| 📺 追剧功能 | 支持 `watchlist.json` 配置追剧列表，自动匹配更新 |
| 🔗 双路由链接 | 电影直达播放，剧集支持指定集数播放 |
| 🧹 自动清理 | 自动删除 7 天前的旧报告 |
| ⏰ 定时推送 | 支持 CronCreate 设置每日定时推送 |

---

## 🏗️ 工作流程 | Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│                      MoonTV Daily Elite Feed                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ① 获取源站列表        ② 全量聚合抓取        ③ 数据处理         │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐     │
│  │   Gateway   │ ───► │  CMS Sites  │ ───► │  Process    │     │
│  │  API Sites  │      │  (5+ sites) │      │  Pipeline   │     │
│  └─────────────┘      └─────────────┘      └─────────────┘     │
│                              │                    │             │
│                              ▼                    ▼             │
│                       ┌─────────────┐      ┌─────────────┐     │
│                       │  Concurrent │      │ • Dedup     │     │
│                       │  Fetching   │      │ • Classify  │     │
│                       │  (all pages)│      │ • Filter    │     │
│                       └─────────────┘      │ • Score     │     │
│                                            │ • Sort      │     │
│                                            └─────────────┘     │
│                                                    │             │
│  ④ 追剧匹配                          ⑤ 输出报告   │             │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐     │
│  │  Watchlist  │ ───► │  Match &    │ ───► │  Markdown   │     │
│  │  Config     │      │  Search     │      │  Report     │     │
│  └─────────────┘      └─────────────┘      └─────────────┘     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🚀 快速开始 | Quick Start

### 1. 安装 Claude Code

确保已安装 [Claude Code](https://claude.ai/code) CLI 工具。

### 2. 克隆项目

```bash
git clone https://github.com/doane2002cn/moontv-skill.git
cd moontv-skill
```

### 3. 配置环境变量

复制示例配置并修改：

```bash
cp config/example.env .env
```

编辑 `.env` 文件，设置你的网关地址和播放页地址。

### 4. 运行技能

在 Claude Code 中执行：

```
/moontv
```

---

## ⚙️ 配置说明 | Configuration

### 环境变量 | Environment Variables

| 变量名 | 说明 | 示例值 |
|--------|------|--------|
| `MOONTV_GATEWAY` | 网关地址，返回可用 CMS 源站列表 | `https://moontv-api.12879737.xyz` |
| `MOONTV_PLAY_URL` | 播放页地址，用于生成播放链接 | `https://moontv.dduan2002cn.xyz/play` |
| `WATCHLIST_FILE` | 追剧配置文件路径 | `config/watchlist.json` |

**示例配置**：

```env
# MoonTV 网关地址（返回可用 CMS 源站列表）
MOONTV_GATEWAY=https://moontv-api.12879737.xyz

# MoonTV 播放页地址（用于生成播放链接）
MOONTV_PLAY_URL=https://moontv.dduan2002cn.xyz/play

# 追剧配置文件路径
WATCHLIST_FILE=config/watchlist.json
```

### 网关 API

网关地址返回的 JSON 格式：

```json
{
  "api_sites": [
    {
      "name": "量子资源",
      "api": "https://cj.lziapi.com/api.php/provide/vod/"
    },
    {
      "name": "金鹰资源",
      "api": "https://jyapi.com/api.php/provide/vod/"
    }
  ]
}
```

### CMS API

每个源站的 API 支持以下参数：

```
GET {site.api}?ac=detail&h=24&pg={page}
```

| 参数 | 说明 |
|------|------|
| `ac=detail` | 返回详细信息 |
| `h=24` | 获取最近 24 小时更新 |
| `pg={page}` | 页码（从 1 开始） |

返回字段：

| 字段 | 说明 |
|------|------|
| `vod_id` | 视频 ID |
| `vod_name` | 片名 |
| `type_name` | 分类名（如"动作片"、"国产剧"） |
| `vod_remarks` | 清晰度/更新状态 |
| `vod_douban_score` | 豆瓣评分（0 表示无评分） |
| `vod_hits` | 播放热度 |
| `vod_time` | 入库时间 |
| `vod_year` | 年份 |

---

## 📺 追剧设置 | Watchlist Setup

### 配置文件格式

创建 `config/watchlist.json`：

```json
{
  "watchlist": [
    {"name": "大唐迷雾", "type": "剧集"},
    {"name": "认识的哥哥", "type": "综艺"},
    {"name": "梦魇绝镇", "type": "剧集"}
  ]
}
```

### 字段说明

| 字段 | 必填 | 说明 |
|------|------|------|
| `name` | ✅ | 剧集/综艺名称关键词（模糊匹配 `vod_name`） |
| `type` | ❌ | 辅助过滤（不参与匹配逻辑） |

### 匹配流程

**路径 A — 从已抓取数据中匹配（零成本）**：
- 遍历 watchlist 中的每个关键词
- 在全量数据中模糊匹配 `vod_name`
- 命中则直接返回结果

**路径 B — 搜索 API 兜底**：
- 对路径 A 未命中的项目
- 并发搜索所有源站的搜索接口
- 取第一个返回有效结果的源站数据

---

## 📊 评分算法 | Scoring Algorithm

### 双路径加权评分

**有豆瓣评分**（`vod_douban_score > 0`）：

```
Score = (豆瓣评分 × 0.6) + (归一化热度 × 0.3) + (时效红利 × 0.1)
```

**无豆瓣评分**（`vod_douban_score == 0`）：

```
Score = (归一化热度 × 0.6) + (时效红利 × 0.3) + (基准分 × 0.1)
```

### 各维度计算

| 维度 | 计算方式 |
|------|----------|
| 豆瓣评分 | 直接使用 `vod_douban_score` |
| 归一化热度 | `(vod_hits / 分类内最大 vod_hits) × 10` |
| 时效红利 | `vod_time` 在最近 12 小时内 → 10，否则 → 0 |
| 基准分 | 固定值 5.0 |

### 权重常量

```python
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

---

## 📋 使用方法 | Usage

### 手动执行

在 Claude Code 中输入：

```
/moontv
```

### 定时推送

使用 Claude Code 的 CronCreate 功能设置每日定时推送：

```
CronCreate:
  cron: "0 9 * * *"
  prompt: "/moontv"
  recurring: true
  durable: true
```

这将在每天上午 9:00 自动执行本技能。

---

## 📄 示例输出 | Example Output

```markdown
# 🎬 MoonTV 每日精选 [2026-06-01]

> 数据源：全量聚合（5 站）| 原始 766 条 → 去重 393 条 → 精选 20 部
>
> 各站数据：量子资源(175) | 金鹰资源(171) | 豪华资源(171) | 暴风资源(132) | 索尼资源(117)

---

## 📋 追剧更新（2/3 有更新）

| 剧名 | 最新进度 | 豆瓣 | 播放 |
|------|---------|------|------|
| 大唐迷雾 | 第19集 | - | [▶️ 播第19集](play-url) |
| 认识的哥哥 | 20260531期 | ⭐8.9 | [▶️ 播放](play-url) |

> 未更新：梦魇绝镇

---

## 🎞️ 热门电影（Top 5）

1. **《木挽町复仇记》** HD中字
   💡 改编自永井纱耶子的同名小说，本作是以江户歌舞伎町轰动一时的事件为背景的时代剧。
   🔗 [▶️ 播放](play-url)

2. **《明亮的日子》** 正片 ⭐7.6
   💡 春意正浓，尼科劳刚刚年满二十四岁，但他并不想大张旗鼓地举办一场盛大的生日派对。
   🔗 [▶️ 播放](play-url)

...

## 📺 热门剧集（Top 5）
## 🍲 热门综艺（Top 5）
## 🎭 热门短剧（Top 5）
## 🎁 福利专区（Top 5）

---

📊 聚合简报：5 站并发 → 原始 766 条 → 去重 393 条 → 过滤 28 条 → 精选 20 部
```

---

## 📁 项目结构 | Project Structure

```
moontv-skill/
├── SKILL.md                 # Claude Code 技能定义（核心）
├── README.md                # 项目说明文档
├── LICENSE                  # MIT 许可证
├── .gitignore               # Git 忽略规则
└── config/
    ├── watchlist.json       # 追剧配置文件
    └── example.env          # 环境变量示例
```

---

## 🔧 错误处理 | Error Handling

| 场景 | 处理方式 |
|------|---------|
| 网关不可达 | 输出错误信息，提示检查 Worker 状态 |
| 网关返回空列表 | 输出"无可用资源站"，建议稍后重试 |
| CMS API 返回异常/超时 | 跳过该源站，继续其他源站 |
| 某分类数据为 0 条 | 该分类显示"今日暂无更新" |
| watchlist.json 不存在 | 跳过追剧匹配，正常执行后续步骤 |
| 追剧搜索全部无结果 | 显示"未找到匹配内容"，不影响主流程 |

---

## 🤝 贡献指南 | Contributing

欢迎提交 Issue 和 Pull Request！

1. Fork 本仓库
2. 创建你的特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交你的更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 打开一个 Pull Request

---

## 📄 许可证 | License

本项目采用 MIT 许可证 - 详见 [LICENSE](LICENSE) 文件

---

## 📝 更新日志 | Changelog

| 版本 | 日期 | 变更 |
|------|------|------|
| v1.0 | 2026-05-31 | 初始版本：三分类各 Top 5，双路径评分，双路由链接 |
| v1.1 | 2026-06-01 | 新增短剧/福利分类（共五类），优化集数提取正则 |
| v1.2 | 2026-06-01 | 全量聚合：并发抓取所有源站，按 vod_name 去重合并 |
| v1.3 | 2026-05-31 | 追剧功能：支持 watchlist.json 配置，双路径匹配更新 |

---

## 🙏 致谢 | Acknowledgments

- [Claude Code](https://claude.ai/code) - AI 编程助手
- [MoonTV](https://github.com/doane2002cn/moontv-skill) - 影视资源聚合平台
- 所有 CMS 资源站提供者

---

## 📞 联系方式 | Contact

- GitHub: [@doane2002cn](https://github.com/doane2002cn)
- Project Link: [https://github.com/doane2002cn/moontv-skill](https://github.com/doane2002cn/moontv-skill)
