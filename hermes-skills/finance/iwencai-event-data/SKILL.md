---
name: iwencai-event-data
description: 问财(iwencai)事件数据查询 — 通过 SkillHub CLI 查询A股事件数据（公告、新闻、研报、股东变动等）
tags: [iwencai, finance, a-share, stock, event, news]
---

# 问财事件数据查询

## 前置条件

1. 检查是否已安装 Iwencai SkillHub CLI：
```bash
command -v iwencai-skillhub-cli || command -v iwencai || echo "未安装"
```

2. 若未安装，执行安装脚本（仅安装 CLI）：
```bash
curl -fsSL https://www.iwencai.com/skillhub/static/0.0.4/download_and_install.sh | bash
```

3. 安装 `事件数据查询` 技能：
```bash
iwencai-skillhub-cli install 事件数据查询 || iwencai install 事件数据查询
```

4. 确保环境变量已配置：
```
IWENCAI_BASE_URL=https://openapi.iwencai.com
IWENCAI_API_KEY=<用户的API Key>
```

## 使用场景

- 查询个股公告、新闻、研报
- 监控股东增减持、高管变动等事件
- 题材/概念相关事件驱动分析

## 数据源

外部来源：/Users/mac/Documents/own/skills/iwencai/event_data.md
