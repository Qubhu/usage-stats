---
name: usage-stats
description: 查询 Claude Code 指定月份的模型用量数据（token用量、费用、请求次数），默认统计 Anthropic 官方模型（opus/sonnet/haiku 等）。关键词：用量、usage、花费、费用、token、请求次数、ccusage。
---

# Claude Code 用量查询

查询指定时间范围内的模型用量数据（token用量、费用）及请求次数。默认仅统计 **Anthropic 官方模型**，除非用户明确要求包含其他模型。

## 触发条件

当用户提到以下关键词时自动触发：
- 用量/usage/花费/费用 + 查询/统计
- token 用量
- 请求次数
- "这个月用了多少"
- ccusage

## 参数解析

从用户输入中提取以下参数：

| 参数 | 说明 | 默认值 |
|------|------|--------|
| **起始月份** | 起始月份，格式 YYYYMM | 当前月份 |
| **结束月份** | 结束月份，格式 YYYYMM | 同起始月份 |
| **模型** | 要统计的模型范围 | Anthropic 官方模型（见下方规则） |
| **包含非 Anthropic 模型** | 是否包含 glm 等非 Anthropic 模型 | 否（除非用户明确要求） |

如果用户说"4月份以来"，起始月份为 202604，结束月份为当前月份。
如果用户说"今天/本周"，使用 `--since` 传入具体日期（YYYYMMDD）。

### 模型筛选规则

**默认行为**：只统计 Anthropic 官方 Claude 系列模型，即模型名包含 `claude` 的模型。归一化后的模型名包括但不限于：
- `opus-*`（如 opus-4-6）
- `sonnet-*`（如 sonnet-4-5, sonnet-4-6）
- `haiku-*`（如 haiku-4-5）

**排除**：模型名中不包含 `claude` 的（如 `glm-5.1`、`glm-4.5-air`、`glm-5` 等），除非用户明确要求"统计所有模型"或"包含 glm"。

**实现方式**：在 Python 中通过判断归一化前的原始模型名 `modelName` 是否包含 `claude` 来过滤，而非硬编码具体模型列表。

## 执行步骤

### Step 1: 获取 ccusage JSON 数据

```bash
npx ccusage@latest daily --since <SINCE> --until <UNTIL> --offline --json --breakdown 2>&1 | tee ~/ccusage_query.json | head -c 200
```

其中：
- `<SINCE>` = 起始月份第一天，格式 YYYYMMDD（如 20260401）
- `<UNTIL>` = 当前日期，格式 YYYYMMDD（如 20260413）

### Step 2: 按模型汇总 token 用量和费用

使用 Python 解析 JSON，**通过判断原始模型名是否包含 `claude` 来动态过滤 Anthropic 官方模型**（模型名需做归一化处理，去掉 `claude-` 前缀和日期后缀如 `-20251001`、`-20250929`）。

如果用户明确要求包含所有模型（包括非 Anthropic），则不进行过滤。

汇总字段：
- Input Tokens
- Output Tokens
- Cache Creation Tokens
- Cache Read Tokens
- Cost (USD)

输出格式为表格，末尾有 TOTAL 行。

```python
import json

with open('/root/ccusage_query.json') as f:
    data = json.load(f)

INCLUDE_ALL = False  # 用户明确要求"所有模型"时设为 True

models = {}
for day in data['daily']:
    for b in day.get('modelBreakdowns', []):
        name = b['modelName']
        # 默认只统计 Anthropic 官方模型（模型名包含 claude）
        if not INCLUDE_ALL and 'claude' not in name:
            continue
        short = name.replace('claude-','').replace('-20251001','').replace('-20250929','')
        if short not in models:
            models[short] = {'input':0,'output':0,'cache_create':0,'cache_read':0,'cost':0}
        models[short]['input'] += b.get('inputTokens',0)
        models[short]['output'] += b.get('outputTokens',0)
        models[short]['cache_create'] += b.get('cacheCreationTokens',0)
        models[short]['cache_read'] += b.get('cacheReadTokens',0)
        models[short]['cost'] += b.get('cost',0)

def fmt(n):
    if n >= 1e9: return f'{n/1e9:.2f}B'
    if n >= 1e6: return f'{n/1e6:.2f}M'
    if n >= 1e3: return f'{n/1e3:.2f}K'
    return str(n)

print("{:<15} {:>12} {:>12} {:>12} {:>12} {:>12}".format(
    'Model', 'Input', 'Output', 'CacheCreate', 'CacheRead', 'Cost(USD)'))
print('-'*78)

grand = {'input':0,'output':0,'cache_create':0,'cache_read':0,'cost':0}
for m in sorted(models.keys()):
    d = models[m]
    cost_s = "$%.2f" % d["cost"]
    print("{:<15} {:>12} {:>12} {:>12} {:>12} {:>12}".format(
        m, fmt(d["input"]), fmt(d["output"]), fmt(d["cache_create"]),
        fmt(d["cache_read"]), cost_s))
    for k in grand: grand[k] += d[k]

print('-'*78)
cost_s = "$%.2f" % grand["cost"]
print("{:<15} {:>12} {:>12} {:>12} {:>12} {:>12}".format(
    'TOTAL', fmt(grand["input"]), fmt(grand["output"]),
    fmt(grand["cache_create"]), fmt(grand["cache_read"]), cost_s))
```

### Step 3: 统计请求次数

从 JSONL 文件中统计 `type: "assistant"` 的记录数，按天汇总。筛选时间范围与 Step 1 的 `--since/--until` 一致。

```bash
find ~/.claude/projects/ -name "*.jsonl" -newermt "<SINCE_DATE>" ! -newermt "<UNTIL_DATE+1day>" -exec grep '"type":"assistant"' {} \; | python3 -c "
import sys, json
from collections import Counter
counts = Counter()
for line in sys.stdin:
    try:
        obj = json.loads(line)
        ts = obj.get('timestamp','')
        if ts:
            counts[ts[:10]] += 1
    except: pass
for d in sorted(counts):
    print(f'{d}: {counts[d]}')
print(f'Total: {sum(counts.values())}')
"
```

其中 `<SINCE_DATE>` 格式为 "YYYY-MM-DD"，`<UNTIL_DATE+1day>` 为结束日期的第二天。

**注意**：find 的 `-newermt` 是左闭右开，所以结束日期需要 +1 天才能包含当天。

### Step 4: 输出汇总

以 markdown 表格形式展示：
1. **模型用量表**：各模型的 Input/Output/Cache Create/Cache Read/Cost 及总计
2. **请求次数表**：按天的请求次数及总请求次数

## 注意事项

- **默认只统计 Anthropic 官方模型**：通过判断模型名是否包含 `claude` 动态过滤，无需硬编码模型列表
- 非 Anthropic 模型（如 `glm-5.1`、`glm-4.5-air`、`glm-5` 等）默认排除，除非用户明确要求包含
- ccusage 的 JSON 输出中费用字段名为 `cost`（不是 `totalCost`）
- JSONL 中的 `type: "assistant"` 记录代表一次模型响应，对应一次请求
- 模型名需要归一化（去掉 `claude-` 前缀和版本日期后缀）

## User-Invocable

This skill can be invoked with `/usage-stats` to query Claude Code usage statistics.
