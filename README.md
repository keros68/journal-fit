# Journal-Aware Revision

> An interactive, requirement-driven diagnostic tool for aligning your manuscript with a target SCI journal's guidelines and stylistic conventions.
>
> 交互式论文诊断工具——先问清楚要查什么，再按需对照期刊要求与同刊惯例，给出修改方向。

---

## What It Does / 做什么

**Only two things: diagnose + suggest directions.** It does NOT write for you, generate replacement text, or produce writing templates.

**只做两件事：检查 + 给修改方向。** 不帮写、不生成替换文字、不做写作模板。

| Feature / 功能 | Description / 说明 |
|---|---|
| 📋 Format Compliance / 格式合规 | Word limits, abstract format, keyword count, highlights, figure requirements, reference style — checked against **real** Author Guidelines, not generic defaults. 字数/摘要/关键词/Highlights/图表/参考文献格式——对照**真实** Guidelines 逐项检查 |
| 📊 Style Alignment / 风格对齐 | Abstract length, reference count, citation density, keyword count — benchmarked against 10-12 same-topic papers from the same journal via OpenAlex + Semantic Scholar + CrossRef. 摘要长度/引用密度/关键词数量——与同刊同主题范文对比 |
| 🔍 Language Style / 语言风格 | Passive voice ratio, sentence length, hedging frequency — requires OA full-text papers (optional). 被动语态/句长/hedging 频率（需 OA 全文范文） |
| 🎯 Section-Specific / 特定部分对标 | Focus on one section (e.g., abstract / introduction / conclusion) and compare its structure with same-journal exemplars. 聚焦某一章节与同刊范文做结构对比 |

## How It Works / 工作流程

```
Phase 0: Clarify what the user actually needs (don't just run everything)
  ↓
Phase 1: Fetch real Author Guidelines (3-layer fallback, 100% publisher coverage)
  + Fetch journal metadata (IF, quartile, review cycle via LetPub)
  ↓
Phase 2: Parse the manuscript draft (.docx/.pdf/.md/.txt)
  ↓
Phase 3: Run only the checks the user asked for
  ↓
Phase 4: Output prioritized suggestions (P0 must-fix / P1 recommended / P2 optional)
```

```
Phase 0: 先问清楚用户到底要查什么（不无脑全跑）
  ↓
Phase 1: 抓取真实 Author Guidelines（三层 fallback，100% 出版商覆盖）
  + 期刊基础信息（IF/分区/审稿周期，经 LetPub）
  ↓
Phase 2: 解析论文草稿（.docx/.pdf/.md/.txt）
  ↓
Phase 3: 只跑用户选择的那几项检查
  ↓
Phase 4: 按优先级输出建议（P0 必改 / P1 建议 / P2 可选）
```

## Why Not Just "Use ChatGPT"? / 为什么不直接问 ChatGPT？

1. **Real Guidelines, not hallucinated defaults.** We tested generic LLM assumptions against actual STOTEN Author Guidelines — **6 out of 13 checks were wrong** (e.g., "Graphical Abstract: optional" → actually required; "references: Vancouver style" → actually any format at submission; "abstract ≤ 250 words" → actually ≤ 300). This tool fetches the real thing.

   **真实 Guidelines，不是编的。** 我们用通用默认值和 STOTEN 真实 Guidelines 做了对比测试，13 项中 6 项出错。这个工具抓的是真实页面。

2. **Empirical style benchmarks.** Style suggestions are backed by actual data from 10-12 same-topic papers in the target journal — with confidence levels and sample sizes stated explicitly.

   **有数据支撑的风格建议。** 每条建议都基于同刊同主题范文的实际统计数据，标注样本量和置信度。

3. **Doesn't write for you.** It diagnoses problems and points you in a direction. The writing stays yours.

   **不替你写。** 只诊断问题、给方向，文字始终是你自己的。

## Quick Setup / 快速配置

### Zero Config (works out of the box) / 零配置（装了就能用）

| Component | API | Cost |
|---|---|---|
| Paper search | OpenAlex | Free, no key |
| Abstract data | Semantic Scholar | Free, no key (~1 req/s) |
| Paper metadata | CrossRef | Free, no key |
| OA full text | Europe PMC | Free, no key |
| Guidelines (Elsevier/MDPI) | Jina Reader | Free, no key |

**Guidelines coverage without Tavily: ~50%** (Elsevier/MDPI/Springer partial; Wiley/ACS/T&F not supported).

### Minimal Config (1 free key → 100% coverage) / 最小配置（1 个免费 key → 100% 覆盖）

1. Register at [tavily.com](https://tavily.com) (free, 1000 req/month)
2. Add the key:

```json
// ~/.claude/settings.json
{
  "env": {
    "TAVILY_API_KEY": "tvly-your-key-here"
  }
}
```

**Guidelines coverage with Tavily: 100%** (Elsevier ✅ / Springer Nature ✅ / MDPI ✅ / Wiley ✅ / ACS ✅ / Taylor & Francis ✅)

### Optional / 可选

```bash
pip install python-docx pymupdf   # .docx and .pdf parsing
# .md/.txt need nothing extra
```

## Guidelines Fetching — 3-Layer Fallback / Guidelines 获取——三层降级策略

Tested across 6 publishers (2026-06-27):

| Publisher | Layer 1 (Tavily extract) | Layer 2 (Jina Reader) | Layer 3 (Tavily search → 3rd party) |
|---|---|---|---|
| Elsevier | ✅ 83-108K chars | ✅ 62K | — |
| MDPI | ✅ 108K | ✅ 108K | — |
| Springer Nature | ⚠️ 13K (mostly nav) | ⚠️ 4K | ✅ via manusights.com |
| Wiley | ❌ | ❌ | ✅ via manusights.com |
| ACS | ❌ | ❌ | ✅ via manusights.com |
| Taylor & Francis | ❌ | ❌ | ✅ via scispace.com |

**Result: 6/6 publishers covered, no manual paste needed.**

See [`references/guidelines-fallback-matrix.md`](references/guidelines-fallback-matrix.md) for full test data and curl commands.

## File Structure / 文件结构

```
journal-aware-revision/
├── SKILL.md                                    # Main skill instructions
├── README.md                                   # This file
└── references/
    ├── guidelines-fallback-matrix.md           # 6-publisher × 4-method test matrix
    └── pitfalls-and-solutions.md               # 8 real-world pitfalls from testing
```

## Design Principles / 设计原则

1. **Requirement-driven, not pipeline-driven.** Ask what the user needs first, then run only relevant steps.
2. **No hallucination.** Every number in the Guidelines checklist must come from a real fetched page. Every style benchmark must come from real API data.
3. **Transparency.** State sample sizes and confidence levels. Flag data gaps honestly.
4. **Doesn't write for you.** Diagnose + suggest direction only.

## Limitations / 局限性

- Does NOT assess scientific correctness, experimental design, or novelty
- Language style checks depend on OA full-text availability (often limited)
- Third-party Guidelines data (Layer 3) may lag behind official updates
- Final academic quality and writing are the author's responsibility

## Tested With / 测试过的期刊

| Journal | Publisher | Field |
|---|---|---|
| Science of the Total Environment | Elsevier | Environmental science |
| Water Research | Elsevier | Environmental engineering |
| J. of Hazardous Materials | Elsevier | Materials/chemistry |
| Nature Climate Change | Springer Nature | Climate science |
| Water (MDPI) | MDPI | Water science |
| Global Change Biology | Wiley | Ecology |
| Environmental Science & Technology | ACS | Environmental science |
| Arid Land Research and Management | Taylor & Francis | Arid land management |

## License

MIT

---

Built for [Hermes Agent](https://hermes-agent.nousresearch.com). Version 3.4.0.
