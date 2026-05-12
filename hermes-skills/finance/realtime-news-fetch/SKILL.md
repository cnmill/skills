---
name: realtime-news-fetch
description: "实时新闻获取与沉淀 — 从 NewsNow 聚合站抓取财经快讯，落库 MongoDB，支持本地优先查询减少外部依赖。"
version: 1.0.0
tags: [finance, news, realtime, mongodb, data-sink]
metadata:
  hermes:
    tags: [finance, news, realtime, mongodb, data-sink]
    related_skills: [china-equity-research]
---

# 实时新闻获取与沉淀

## 触发条件

- 需要获取当日/近期财经新闻辅助市场判断
- 每日复盘需要新闻事件背景
- 用户问"今天有什么新闻"、"最近有什么政策"
- 产业链研究需要最新动态

## 数据源

### NewsNow 聚合站（主源）

- 页面地址: `https://newsnow.busiyi.world/c/realtime`
- 特点: JS渲染（React），必须用 browser 工具访问
- 内容源: 华尔街见闻快讯、联合早报、36氪等
- API端点（不稳定，备用）:
  - `/api/s?id=wallstreetcn-quick` — 华尔街见闻快讯
  - `/api/s?id=36kr-quick` — 36氪快讯
  - `/api/s?id=zaobao` — 联合早报
  - 需要 Referer: `https://newsnow.busiyi.world/` 和 User-Agent

### 抓取方法

```
1. browser_navigate("https://newsnow.busiyi.world/c/realtime")
2. browser_snapshot(full=true) 获取新闻列表
3. 提取标题、时间、来源
4. 按关键词过滤市场相关新闻
5. 落库 MongoDB
```

## MongoDB 数据沉淀

### 集合: `news_realtime`

数据库: `tradingagentscn_v0_mac`（与 TradingAgents-CN 共用）

```javascript
// 文档结构
{
  "_id": ObjectId,
  "title": "央行宣布降准0.5个百分点",
  "source": "wallstreetcn",  // wallstreetcn | zaobao | 36kr | other
  "pub_time": ISODate("2026-05-12T15:30:00Z"),  // 发布时间
  "fetch_time": ISODate("2026-05-12T16:00:00Z"), // 抓取时间
  "category": "macro",  // macro | market | industry | company | global
  "keywords": ["央行", "降准", "货币政策"],
  "relevance_score": 0.9,  // 与A股相关度 0-1
  "url": "https://...",  // 原文链接（如有）
  "summary": ""  // 可选，AI摘要
}
```

### 索引

```javascript
db.news_realtime.createIndex({"pub_time": -1})
db.news_realtime.createIndex({"fetch_time": -1})
db.news_realtime.createIndex({"title": 1}, {unique: true})  // 去重
db.news_realtime.createIndex({"category": 1, "pub_time": -1})
db.news_realtime.createIndex({"keywords": 1})
```

### 分类规则

| category | 关键词 |
|----------|--------|
| macro | 央行、降息、降准、GDP、CPI、PMI、财政部、发改委、国务院 |
| market | A股、沪指、涨停、跌停、北向、外资、成交额、IPO |
| industry | 半导体、芯片、AI、算力、新能源、光伏、锂电、汽车 |
| company | 具体公司名、业绩、减持、回购、分红 |
| global | 美联储、关税、美股、原油、黄金、地缘 |

## 查询优先级策略

**核心原则: 本地优先，外部补充**

```
1. 先查 MongoDB news_realtime 集合
   - 条件: fetch_time >= 当日 00:00
   - 如果有 >= 10 条当日新闻 → 直接使用，不再外部抓取
2. 如果本地数据不足（< 10 条或无当日数据）
   - 用 browser 访问 NewsNow 抓取
   - 抓取后立即落库
3. 复盘时合并: 本地已有 + 新抓取（去重）
```

### 查询脚本

```python
from pymongo import MongoClient
from datetime import datetime, timedelta

client = MongoClient("mongodb://localhost:27017")
db = client["tradingagentscn_v0_mac"]
col = db["news_realtime"]

# 查询当日新闻
today_start = datetime.now().replace(hour=0, minute=0, second=0, microsecond=0)
news = list(col.find(
    {"fetch_time": {"$gte": today_start}},
    sort=[("pub_time", -1)]
).limit(50))

print(f"当日已沉淀新闻: {len(news)} 条")
for n in news[:20]:
    print(f"  [{n['category']}] {n['title']}")
```

### 写入脚本

```python
from pymongo import MongoClient, UpdateOne
from datetime import datetime

client = MongoClient("mongodb://localhost:27017")
db = client["tradingagentscn_v0_mac"]
col = db["news_realtime"]

def save_news(news_list):
    """批量写入新闻（upsert去重）"""
    ops = []
    for item in news_list:
        ops.append(UpdateOne(
            {"title": item["title"]},
            {"$setOnInsert": item},
            upsert=True
        ))
    if ops:
        result = col.bulk_write(ops)
        print(f"新增: {result.upserted_count}, 已存在: {len(ops) - result.upserted_count}")
```

## 与复盘 cron job 的集成

复盘 agent 工作流:
1. 运行 `a-share-daily-fetch.py` 获取行情数据
2. 查询 MongoDB `news_realtime` 获取当日已沉淀新闻
3. 如果新闻不足，browser 访问 NewsNow 补充抓取并落库
4. 综合行情+新闻生成复盘报告

## 注意事项

- NewsNow API 使用 Cloudflare D1 数据库，高峰期容易过载返回500
- 页面是 JS 渲染，curl 无法直接获取内容，必须用 browser
- 新闻标题作为唯一键去重，避免重复入库
- pub_time 尽量从页面"X分钟前"推算为绝对时间
- 定期清理 30 天前的新闻数据（可选）
