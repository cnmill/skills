---
name: iwencai-base
description: 问财(iwencai)基本资料查询 — 通过 SkillHub CLI 查询A股公司基本资料（股票代码、公司名称、行业分类、主营业务等）
tags: [iwencai, finance, a-share, stock, fundamental]
---

# 问财基本资料查询

## 前置条件

1. 检查是否已安装 Iwencai SkillHub CLI：
```bash
command -v iwencai-skillhub-cli || command -v iwencai || echo "未安装"
```

2. 若未安装，执行安装脚本（仅安装 CLI）：
```bash
curl -fsSL https://www.iwencai.com/skillhub/static/0.0.4/download_and_install.sh | bash
```

3. 安装 `基本资料查询` 技能：
```bash
iwencai-skillhub-cli install 基本资料查询 || iwencai install 基本资料查询
```

4. 确保环境变量已配置：
```
IWENCAI_BASE_URL=https://openapi.iwencai.com
IWENCAI_API_KEY=<用户的API Key>
```

## 使用场景

- 查询个股基本信息（代码、名称、行业、主营）
- 批量获取板块成分股基本资料
- 产业链分析中的公司基础数据获取

## 数据源

外部来源：/Users/mac/Documents/own/skills/iwencai/base.md
