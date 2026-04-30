---
name: us-stock-data-sync
description: 美股数据同步入库 — 通过东方财富 datacenter API 获取纽交所+纳斯达克全量美股基础信息，绕过代理和限速问题，写入 MongoDB。
tags: [us-stock, eastmoney, mongodb, data-sync, akshare]
triggers:
  - 美股数据同步
  - 获取美股列表
  - US stock sync
  - 入库美股
  - 拉取美股数据
---

# 美股数据同步入库

## 背景

将纽交所 (NYSE) + 纳斯达克 (NASDAQ) 的美股基础信息批量入库到 MongoDB，用于后续投研分析。

## 数据源选择（踩坑记录）

| 方案 | 问题 | 结论 |
|------|------|------|
| AKShare `stock_us_spot_em` | macOS 系统代理 (verge-mihomo 127.0.0.1:7897) 拦截 push2.eastmoney.com，Python requests 读取系统代理配置后连接失败，monkey-patch Session.trust_env 也无效 | ❌ 不可用 |
| AKShare `stock_us_spot` / `get_us_stock_name` (新浪源) | 861 页逐页爬取，每页约 1 秒，总耗时 15+ 分钟 | ❌ 太慢 |
| yfinance | Yahoo Finance 频繁限速 (YFRateLimitError)，详细数据拿不到 | ❌ 不稳定 |
| **curl + 东方财富 datacenter API** | `curl --noproxy '*'` 绕过系统代理直连，API 稳定快速 | ✅ 推荐 |

## 核心方法：curl + 东方财富 datacenter API

### API 端点

```
https://datacenter-web.eastmoney.com/api/data/v1/get
```

### 关键参数

```
reportName=RPT_USF10_INFO_ORGPROFILE    # 美股公司概况表
columns=SECUCODE,SECURITY_CODE,SECURITY_NAME_ABBR,ORG_EN_ABBR,BELONG_INDUSTRY,BELONG_MARKET
sortColumns=SECURITY_CODE
sortTypes=1
pageSize=500                             # 每页最大 500
pageNumber={page}
source=SECURITIES
client=PC
```

### 可用字段（columns=ALL 时返回）

- `SECUCODE` — 带交易所后缀的代码，如 `AAPL.O`（O=纳斯达克, N=纽交所, A=美交所, F=场外）
- `SECURITY_CODE` — 纯代码，如 `AAPL`
- `SECURITY_NAME_ABBR` — 中文简称
- `ORG_EN_ABBR` — 英文全称
- `BELONG_INDUSTRY` — 行业分类（中文，156 个行业）
- `BELONG_MARKET` — 交易所（纽交所/纳斯达克/场外柜台）
- `EMP_NUM` — 员工数
- `ORG_PROFILE` — 公司简介
- `FOUND_DATE` — 成立日期
- `ORG_WEB` — 官网

### 响应格式

```json
{
  "result": {
    "pages": 43,
    "count": 21476,
    "data": [{"SECUCODE": "AAPL.O", "SECURITY_CODE": "AAPL", ...}]
  },
  "success": true
}
```

## 步骤

### 1. 分页拉取数据（curl 绕过代理）

```python
import subprocess, json

all_stocks = []
for page in range(1, 100):
    url = (
        f'https://datacenter-web.eastmoney.com/api/data/v1/get'
        f'?sortColumns=SECURITY_CODE&sortTypes=1'
        f'&pageSize=500&pageNumber={page}'
        f'&reportName=RPT_USF10_INFO_ORGPROFILE'
        f'&columns=SECUCODE,SECURITY_CODE,SECURITY_NAME_ABBR,ORG_EN_ABBR,BELONG_INDUSTRY,BELONG_MARKET'
        f'&source=SECURITIES&client=PC'
    )
    result = subprocess.run(
        ['curl', '-s', '--noproxy', '*', '--connect-timeout', '5', '--max-time', '10', url],
        capture_output=True, text=True, timeout=15
    )
    if not result.stdout.strip():
        break
    data = json.loads(result.stdout)
    if not data.get('success') or not data['result'].get('data'):
        break

    for item in data['result']['data']:
        market = item.get('BELONG_MARKET', '')
        if market in ('纽交所', '纳斯达克'):  # 过滤掉场外柜台
            all_stocks.append(item)

    total_pages = data['result']['pages']
    if page >= total_pages:
        break

# 约 43 页，耗时 ~15 秒，得到 ~5900 只 NYSE+NASDAQ 美股
```

### 2. 写入 MongoDB

```python
import pymongo
from datetime import datetime

client = pymongo.MongoClient('mongodb://localhost:27017/')
db = client['tradingagentscn_v0_mac']
coll = db['stock_basic_info_us']

docs = []
market_map = {'O': 'NASDAQ', 'N': 'NYSE', 'A': 'AMEX'}
for s in all_stocks:
    secucode = s.get('SECUCODE', '')
    suffix = secucode.split('.')[-1] if '.' in secucode else ''
    docs.append({
        'code': s['SECURITY_CODE'],
        'secucode': secucode,
        'name': s.get('SECURITY_NAME_ABBR', ''),
        'name_en': s.get('ORG_EN_ABBR', '') or s['SECURITY_CODE'],
        'industry': s.get('BELONG_INDUSTRY', '') or '',
        'market': 'US',
        'exchange': market_map.get(suffix, s.get('BELONG_MARKET', '')),
        'area': 'US',
        'source': 'eastmoney_datacenter',
        'currency': 'USD',
        'pe': None, 'pb': None, 'total_mv': None, 'close': None,
        'sector': '',
        'updated_at': datetime.utcnow()
    })

coll.delete_many({})
for i in range(0, len(docs), 500):
    coll.insert_many(docs[i:i+500])

coll.create_index('code', unique=True)
coll.create_index('exchange')
coll.create_index('industry')
```

### 3. 验证

```python
total = coll.count_documents({})
nasdaq = coll.count_documents({'exchange': 'NASDAQ'})
nyse = coll.count_documents({'exchange': 'NYSE'})
print(f"总计: {total}, NASDAQ: {nasdaq}, NYSE: {nyse}")
# 预期: ~5926 总计, ~3932 NASDAQ, ~1994 NYSE
```

## Pitfalls

1. **macOS 系统代理干扰**：verge-mihomo 等代理工具在 macOS 系统偏好设置中配置了 HTTP/HTTPS 代理 (127.0.0.1:7897)，Python urllib/requests 会自动读取。`curl --noproxy '*'` 可以绕过，但 Python 内 monkey-patch `trust_env=False` 对 akshare 内部 session 不一定生效。最可靠的方式是用 subprocess 调 curl。
2. **东方财富 push2 API vs datacenter API**：push2.eastmoney.com 的实时行情 API 在代理环境下 SSL 握手失败；datacenter-web.eastmoney.com 的数据中心 API 更稳定。
3. **SECUCODE 后缀映射**：`.O` = NASDAQ, `.N` = NYSE, `.A` = AMEX, `.F` = 场外柜台 (OTC)。
4. **filter 参数中文编码**：datacenter API 的 filter 参数传中文时 URL 编码可能出问题，建议在本地 Python 中过滤而非在 API 参数中过滤。
5. **全量数据约 21000+**：包含大量场外柜台 (OTC) 小票，通常只需纽交所+纳斯达克的 ~5900 只。
6. **BRK.B 等特殊代码**：伯克希尔 B 股在东方财富中代码可能不同（如 BRK_B 或 BRK.B），需单独处理。

## 扩展：补充估值数据

如需 PE/PB/市值等估值数据，可探索东方财富其他报表：
- 尝试 `reportName` 中搜索 `RPT_USSK_` 或 `RPT_USF10_` 前缀的报表
- 或用 AKShare 的 `stock_us_valuation_baidu`（百度源）逐只补充
- 或用 `stock_financial_us_analysis_indicator_em` 获取财务指标

## 数据规模参考（2026-04）

- 纽交所: ~1994 只
- 纳斯达克: ~3932 只
- 合计: ~5926 只
- 行业分类: 156 个
- Top 行业: 投资银行业与经纪业(710), 生物科技(396), 制药(281), 应用软件(221)
