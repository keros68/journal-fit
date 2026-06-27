# Guidelines 自动获取方案矩阵

基于 2026-06-27 跨出版商实测（6 家出版商 × 4 种抓取方案）。这是一个 dated snapshot，用于选择 fallback 顺序；实际使用时仍需重新验证当前页面是否可抓取。

## 测试矩阵

| 出版商 | 代表期刊 | Tavily extract | Markdown reader cascade | Tavily search→第三方 | 推荐层 |
|--------|---------|---------------|-------------|---------------------|--------|
| Elsevier | STOTEN, Water Research, JHM | ✅ (83K-108K) | ✅ (62K) | — | Layer 1 |
| MDPI | Water | ✅ (108K) | ✅ (108K) | — | Layer 1/2 |
| Springer Nature | Nature Climate Change | ⚠️ (13K 但正文空) | ⚠️ (4K 导航为主) | ✅ (manusights) | Layer 3 |
| Wiley | Global Change Biology | ❌ (fetch fail) | ❌ (bot check) | ✅ (manusights 33K) | Layer 3 |
| ACS | ES&T | ❌ (fetch fail) | ❌ (cookie wall) | ✅ (manusights 22K) | Layer 3 |
| Taylor & Francis | Arid Land Res. & Mgmt. | ❌ (fetch fail) | ⚠️ (Jina 波动；复测可拿到 47K 官方说明) | ✅ (scispace 17K) | Layer 2/3 |

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

### Layer 2: Built-in Markdown reader cascade（无需额外 skill）

This layer is a lightweight URL-to-Markdown fallback built into `journal-fit`. Do not require users to install another skill.

Try these in order:

```bash
curl -s "https://r.jina.ai/{url}"
# 可选 header: X-Engine: browser, X-Return-Format: text
```

If Jina returns an error, security page, cookie page, or near-empty output:

```bash
curl -s "https://defuddle.md/{url}"
```

If both hosted readers fail and `npx` is already available, try:

```bash
npx --yes agent-fetch "{url}" --json
```

- Jina 和 defuddle 免费无需 key
- `agent-fetch` is optional; never make Node/npm setup a prerequisite for novice users
- Wiley 返回 "Performing security verification" → bot check 拦截
- ACS 常只返回入口页或 cookie 页面，需要继续追踪官方 Author Guidelines 链接
- Taylor & Francis 页面波动明显：曾返回空，也曾通过 Jina 拿到完整官方说明
- Nature/Springer 往往先返回 hub 页面；需要继续访问官方二级链接，如 initial formatting、preparing your submission

**Content validation before accepting output:**

- Must contain substantial body text, not just title/navigation.
- Must include guideline signals such as `abstract`, `manuscript`, `word`, `figure`, `reference`, `submission`, or `author guidelines`.
- Reject pages dominated by `security verification`, `cookie`, `Access Denied`, `404`, login prompts, or generic publisher navigation.
- If the official page is a hub, follow relevant official subpage links before using third-party summaries.

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
  -H "Content-Type: application/json" \
  -d '{"urls":["{manusights_url}"], "extract_depth":"advanced"}'
```

**第三方聚合站效果：**

| 站点 | 测试期刊 | 提取量 | 数据质量 |
|------|---------|--------|---------|
| manusights.com | GCB (Wiley) | 33K chars | 高：结构化表格，逐项列出字数/摘要/图表/参考文献/Highlights |
| manusights.com | ES&T (ACS) | 22K chars | 高：按稿件类型分列（Research Article 7000词/摘要150词等） |
| scispace.com | Arid Land (T&F) | 17K chars | 中：格式要求摘要，不如 manusights 详细 |

**Tavily search 的 AI answer 可作为检索线索，但不要作为唯一证据：**
- 可用来定位关键词、第三方聚合页或官方说明入口
- 数值型要求仍应尽量从官方页、抽取到的页面正文、或明确标注的第三方页面中复核
- 若只能使用 AI answer，输出时必须标注低置信度和来源限制

**风险：** 第三方数据可能滞后于官方。输出时标注数据来源。

## 执行流程（agent 视角）

```
1. 根据 ISSN/刊名判断出版商
2. 构造官方 Guidelines URL
3. 调 Layer 1 (Tavily extract；仅在 TAVILY_API_KEY 已配置时)
4. 检查返回内容是否包含正文关键词（abstract/manuscript/word/figure/reference）
   - 有正文 → 提取 Guidelines Checklist，继续
   - 无正文或失败 → 进入 Layer 2
5. 调 Layer 2 (built-in Markdown reader cascade: Jina → defuddle → optional agent-fetch)
   - 有正文 → 提取，继续
   - 失败 → 进入 Layer 3
6. 调 Layer 3 (Tavily search → 第三方 extract)
   - 有结果 → 提取，标注"数据来源：第三方"
   - 失败 → Layer 4 手动粘贴
```

## 覆盖率总结

- Layer 1 单独覆盖率：3/6 出版商（Elsevier, MDPI, 部分 Springer）
- Layer 1+2 覆盖率：优于单一 Jina，但仍会受 publisher 反爬和页面结构影响
- Layer 1+2+3：在这批样本中覆盖 **6/6 出版商**
- Layer 4 手动粘贴：理论兜底；这批实测中未使用

**结论：三层 fallback 能显著提高自动获取成功率，但不是永久覆盖承诺。输出中必须标注来源类型；第三方或搜索摘要结果需要提醒用户投稿前复核官方最新 Guidelines。**
