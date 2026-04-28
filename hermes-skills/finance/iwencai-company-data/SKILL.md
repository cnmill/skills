---
name: iwencai-company-data
description: 问财(iwencai)公司经营数据查询 — 通过 SkillHub CLI 查询A股公司经营数据（营收、利润、ROE、毛利率等财务指标）
tags: [iwencai, finance, a-share, stock, financial, operating]
---

# 问财公司经营数据查询

## 前置条件

1. 检查是否已安装 Iwencai SkillHub CLI：
```bash
command -v iwencai-skillhub-cli || command -v iwencai || echo "未安装"
```

2. 若未安装，执行安装脚本（仅安装 CLI）：
```bash
curl -fsSL https://www.iwencai.com/skillhub/static/0.0.4/download_and_install.sh | bash
```

3. 安装 `公司经营数据查询` 技能：
```bash
iwencai-skillhub-cli install 公司经营数据查询 || iwencai install 公司经营数据查询
```

4. 确保环境变量已配置：
```
IWENCAI_BASE_URL=https://openapi.iwencai.com
IWENCAI_API_KEY=<用户的API Key>
```

## 使用场景

- 查询公司财务报表数据（营收、净利润、毛利率等）
- 筛选财务指标优质公司（高ROE、低PE等）
- 行业内公司经营数据横向对比

## 数据源

外部来源：/Users/mac/Documents/own/skills/iwencai/company_operating_data.md
