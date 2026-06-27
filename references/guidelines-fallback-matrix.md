# Guidelines 自动获取方案矩阵

基于 2026-06-27 跨出版商实测（6 家出版商 × 4 种抓取方案）。

## 测试矩阵

| 出版商 | 代表期刊 | Tavily extract | Jina Reader | Tavily search→第三方 | 推荐层 |
|--------|---------|---------------|-------------|---------------------|--------|
| Elsevier | STOTEN, Water Research, JHM | ✅ (83K-108K) | ✅ (62K) | — | Layer 1 |
| MDPI | Water | ✅ (108K) | ✅ (108K) | — | Layer 1/2 |
| Springer Nature | Nature Climate Change | ⚠️ (13K 但正文空) | ⚠️ (4K 导航为主) | ✅ (manusights) | Layer 3 |
| Wiley | Global Change Biology | ❌ (fetch fail) | ❌ (bot check) | ✅ (manusights 33K) | Layer 3 |
| ACS | ES&T | ❌ (fetch fail) | ❌ (cookie wall) | ✅ (manusights 22K) | Layer 3 |
| Taylor & Francis | Arid Land Res. & Mgmt. | ❌ (fetch fail) | ❌ (0 chars) | ✅ (scispace 17K) | Layer 3 |

## 各层详细说明

### Layer 1: Tavily extract（直抓官方 Guidelines 页）

```bash
curl -s -X POST "https://api.tavily.com/extract" \
  -H "Authorization: Bearer $TAVILY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"urls":["{url}"], "extract_depth":"advanced", "timeout":60}'
```

- Elsevier: 一次拿到 83K-108K chars 完整 Guidelines，可直接 regex 提取所有格式要求
- MDPI: 同样完整
- Springer Nature: 返回 13K chars 但大部分是导航菜单/Cookie 声明，正文几乎没有
- Wiley/ACS/T&F: Tavily 返回 `failed_results`（"Failed to fetch url"）

### Layer 2: Jina Reader (r.jina.ai)

```bash
curl -s "https://r.jina.ai/{url}"
# 可选 header: X-Engine: browser, X-Return-Format: text
```

- 免费无需 key
- 覆盖范围和 Tavily extract 几乎一样
- Wiley 返回 "Performing security verification" → bot check 拦截
- ACS 返回 cookie consent 页面
- T&F 返回空
- **价值：Tavily key 耗尽时的免费备选，不是不同覆盖范围的补充**

### Layer 3: Tavily search → 第三方聚合站

```bash
# Step 1: search 找到关键信息 + 第三方 URL
curl -s -X POST "https://api.tavily.com/search" \
  -H "Authorization: Bearer $TAVILY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query":"{期刊名} author guidelines word limit abstract figures", "search_depth":"advanced", "include_answer":true}'

# Step 2: extract 第三方页面拿全文
curl -s -X POST "https://api.tavily.com/extract" \
  -H "Authorization: Bearer $TAVILY_API_KEY" \
  -d '{"urls":["{manusights_url}"], "extract_depth":"advanced"}'
```

**第三方聚合站效果：**

| 站点 | 测试期刊 | 提取量 | 数据质量 |
|------|---------|--------|---------|
| manusights.com | GCB (Wiley) | 33K chars | 高：结构化表格，逐项列出字数/摘要/图表/参考文献/Highlights |
| manusights.com | ES&T (ACS) | 22K chars | 高：按稿件类型分列（Research Article 7000词/摘要150词等） |
| scispace.com | Arid Land (T&F) | 17K chars | 中：格式要求摘要，不如 manusights 详细 |

**Tavily search 的 AI answer 质量很高，可直接用：**
- "GCB 正文上限15000词，摘要250词结构化（Aim/Location/Time Period/...），图表 ≥300 DPI"
- "ES&T 摘要 ≤150 词，submission system 强制执行，需 TOC graphic"
- "Arid Land 正文 ≤15000 词，摘要 ≤150 词"

**风险：** 第三方数据可能滞后于官方。输出时标注数据来源。

## 执行流程（agent 视角）

```
1. 根据 ISSN/刊名判断出版商
2. 构造官方 Guidelines URL
3. 调 Layer 1 (Tavily extract)
4. 检查返回内容是否包含正文关键词（abstract/manuscript/word/figure/reference）
   - 有正文 → 提取 Guidelines Checklist，继续
   - 无正文或失败 → 进入 Layer 2
5. 调 Layer 2 (Jina Reader)
   - 有正文 → 提取，继续
   - 失败 → 进入 Layer 3
6. 调 Layer 3 (Tavily search → 第三方 extract)
   - 有结果 → 提取，标注"数据来源：第三方"
   - 失败 → Layer 4 手动粘贴
```

## 覆盖率总结

- Layer 1 单独覆盖率：3/6 出版商（Elsevier, MDPI, 部分 Springer）
- Layer 1+2 覆盖率：不变（覆盖范围相同）
- Layer 1+2+3 覆盖率：**6/6 出版商（100%）**
- Layer 4 手动粘贴：理论兜底，实测中不需要

**结论：三层 fallback 可实现全自动 Guidelines 获取，无需用户手动粘贴。**
