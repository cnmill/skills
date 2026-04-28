---
name: a-share-daily-review
description: A股每日大盘复盘 — 获取当日指数/行业/成交数据并生成结构化复盘报告。覆盖数据获取、代理绕过、多源降级、报告模板。
tags: [finance, a-share, daily-review, market-data, index, sector, sina, akshare]
---

# A股每日大盘复盘

## 触发条件

- 用户要求"今日复盘"、"大盘复盘"、"今天行情怎么样"、"收盘总结"
- 用户要求获取当日A股指数、行业板块、涨跌统计等数据
- 定时任务（cron）每日收盘后自动生成复盘

## 数据获取优先级

### 第一选择：akshare（需网络直连东方财富）

```python
import akshare as ak
df = ak.stock_zh_index_spot_em()  # 主要指数
```

**已知问题**：macOS 系统代理（如 Clash 127.0.0.1:7897）会导致 akshare 连接东方财富 push2 接口失败（ProxyError），即使设置 `NO_PROXY=*` 也可能无效，因为 requests 库的 `trust_env=True` 会读取 macOS 系统代理设置（通过 `networksetup` 命令可查看）。

检测代理：
```bash
networksetup -getwebproxy Wi-Fi
networksetup -getsecurewebproxy Wi-Fi
```

### 第二选择（推荐）：新浪财经直连 API

新浪接口支持 `--noproxy '*'` 绕过系统代理直连，且无 User-Agent/Referer 限制（部分接口需 Referer）。

#### 主要指数实时行情

```bash
# 简要格式：名称,最新价,涨跌额,涨跌幅%,成交量(手),成交额(万元)
curl -s --noproxy '*' -H "Referer: https://finance.sina.com.cn" \
  "https://hq.sinajs.cn/list=s_sh000001,s_sz399001,s_sz399006,s_sh000300,s_sh000016,s_sh000905"

# 详细格式：名称,今开,昨收,最新价,最高,最低,...,日期,时间
curl -s --noproxy '*' -H "Referer: https://finance.sina.com.cn" \
  "https://hq.sinajs.cn/list=sh000001,sz399001,sz399006,sh000300,sh000016,sh000905,sh000688"
```

指数代码映射：
- sh000001=上证指数, sz399001=深证成指, sz399006=创业板指
- sh000300=沪深300, sh000016=上证50, sh000905=中证500, sh000688=科创50

简要格式字段顺序：`名称,最新价,涨跌额,涨跌幅%,成交量(手),成交额(万元)`

详细格式字段顺序（逗号分隔）：
- [0]=名称, [1]=今开, [2]=昨收, [3]=最新价, [4]=最高, [5]=最低
- [8]=成交量(股), [9]=成交额(元), [30]=日期, [31]=时间

**注意**：新浪返回的中文是 GBK 编码，在 Python 中解析时可能显示为乱码。用 `iconv -f gbk -t utf-8` 转码，或在 Python 中用正则按位置解析数值字段（不依赖中文名称）。

#### 行业板块涨跌排名

```bash
# 新浪行业板块（GBK编码，需转码）
curl -s --noproxy '*' \
  "https://vip.stock.finance.sina.com.cn/q/view/newSinaHy.php" | iconv -f gbk -t utf-8
```

返回 JS 变量，每条记录格式：
`板块代码,板块名称,成分股数,均价,涨跌额,涨跌幅%,成交量,成交额,领涨股代码,领涨股涨幅,领涨股价格,领涨股涨跌额,领涨股名称`

用正则解析：
```python
pattern = r'"(new_\w+),([^,]+),(\d+),([^,]+),([^,]+),([^,]+),(\d+),(\d+),(\w+),([^,]+),([^,]+),([^,]+),([^"]+)"'
```

#### 概念板块涨跌排名（热点题材必备）

用户要求"热点概念"、"题材复盘"时必须获取此数据。与行业板块不同，概念板块反映市场题材炒作方向。

```bash
# 新浪概念板块（GBK编码，需转码）— 返回 ~130 个概念板块
curl -s --noproxy '*' \
  "https://money.finance.sina.com.cn/q/view/newFLJK.php?param=class" | iconv -f gbk -t utf-8
```

返回 JS 变量 `S_Finance_bankuai_class`，每条记录格式（逗号分隔）：
`概念代码(gn_xxx),概念名称,成分股数,均价,涨跌额,涨跌幅%,成交量,成交额,领涨股代码,领涨股涨幅,领涨股价格,领涨股涨跌额,领涨股名称`

解析要点：
- 概念代码以 `gn_` 开头（区别于行业板块的 `new_` 前缀）
- 按涨跌幅(第6字段)排序即可得到热点概念TOP和跌幅概念TOP
- 领涨股信息在字段9-13，可直接用于报告

#### 概念板块涨跌排名（重要！）

```bash
# 新浪概念板块（GBK编码，需转码）— 包含华为汽车/CRO/氢能源等真正题材概念
curl -s --noproxy '*' \
  "https://money.finance.sina.com.cn/q/view/newFLJK.php?param=class" | iconv -f gbk -t utf-8
```

返回 JS 变量，每条记录格式同行业板块：
`板块代码,板块名称,成分股数,均价,涨跌额,涨跌幅%,成交量,成交额,领涨股代码,领涨股涨幅,领涨股价格,领涨股涨跌额,领涨股名称`

**过滤规则**：输出时需过滤掉非题材类概念（含H股、含B股、含GDR、融资融券、基金重仓、社保重仓、券商重仓、信托重仓、保险重仓、QFII重仓、科创50、业绩预升、业绩预降、准ST股、ST板块、分拆上市、股权激励、本月解禁、送转潜力等），只保留真正的题材概念。

#### 申万行业分类

```bash
curl -s --noproxy '*' \
  "https://vip.stock.finance.sina.com.cn/q/view/newFLJK.php" | iconv -f gbk -t utf-8
```

### 第三选择：东方财富直连（需绕过代理）

东方财富 push2 接口即使绕过代理也可能返回空响应（疑似 User-Agent 或 Referer 校验）。如果必须使用：

```bash
curl -s --noproxy '*' --max-time 10 \
  -H "Referer: https://data.eastmoney.com" \
  -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36" \
  "https://push2.eastmoney.com/api/qt/clist/get?pn=1&pz=20&po=1&np=1&ut=bd1d9ddb04089700cf9c27f6f7426281&fltt=2&invt=2&fid=f3&fs=m:90+t:2&fields=f2,f3,f4,f12,f14"
```

**实测结果**：2026年4月在有系统代理的环境下，东方财富 push2 接口从 curl 和浏览器 JSONP 均返回空/status 0，完全不可用。新浪接口更可靠。

### 港股通净买额（浏览器 DOM 抓取）

东方财富行情中心页面 `https://quote.eastmoney.com/center/boardlist.html#concept_board` 会渲染沪深港通数据。虽然 push2 API 不可用，但页面 DOM 中可读取港股通净买额（如"港股通(沪>港) 净买额 108.32亿"）。用 browser_navigate 打开页面后从 snapshot 中提取即可。

## 复盘报告模板

报告应包含以下模块：

1. **主要指数收盘**：表格列出收盘价、涨跌幅、今开、最高、最低、振幅
2. **成交概况**：沪市/深市/两市合计成交额，港股通净买额
3. **热点概念板块**：概念板块涨幅TOP10（标注核心逻辑/催化剂）和跌幅TOP10（标注回调原因）— 这是用户最关注的题材方向
4. **行业板块**：行业板块涨跌TOP5
5. **盘面特征**：
   - 指数分化情况（大盘vs中小盘vs科创/创业板）
   - 成交量水平判断（缩量/放量/维持）
   - 风格判断（高低切换/题材轮动/一致性上涨下跌）
6. **核心解读**：2-3个最重要的盘面信号及其含义
7. **与研究主线关联**：将当日表现映射到用户关注的研究方向
8. **后续关注**：明日/近期需要跟踪的关键变量

## 存储与推送

- 文件路径：`/Users/mac/Documents/own/hermesfile/research/每日复盘/YYYY-MM-DD-A股大盘复盘.md`
- 写入后立即执行 `git add -A && git commit -m "add YYYY-MM-DD A股大盘复盘" && git push`

## 注意事项

- 4月27日是周日，但A股有时会因调休补班而开市，以实际数据为准
- 北向资金实时数据可能因政策变更不再披露（2026年接口返回全0）
- 涨跌停统计东方财富接口报表名可能变更，需动态适配
- 两市成交额 = 沪市成交额(万) + 深市成交额(万)，注意单位换算
