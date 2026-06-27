---
name: journal-fit
description: Use when aligning a manuscript draft with a target journal before submission, including Author Guidelines compliance, same-journal style benchmarking, section-specific diagnostics, and prioritized revision directions without drafting replacement prose.
---

# Journal Fit

Diagnose whether a manuscript fits a target journal's requirements and conventions. The skill does two things only: check the draft and give revision directions. Do not draft replacement prose, rewrite sections, or create writing templates unless the user explicitly switches to a writing/polishing task.

## Operating Rules

- Ask what to check before running tools. Do not execute the full workflow when the user only needs one diagnostic.
- Use real journal evidence for hard compliance claims. Do not replace missing Author Guidelines with publisher defaults.
- Treat third-party guideline aggregators as fallback evidence only. Label the source and tell the user to verify current official requirements before submission.
- Mark unsupported items as "unable to assess" rather than guessing.
- Keep outputs diagnostic: issue, evidence, risk, and revision direction.
- If the user writes in Chinese, answer in Chinese unless they request another language.

## Start Here

Collect the minimum missing information:

1. Target journal name or ISSN.
2. Manuscript draft path or pasted text. Supported inputs are `.docx`, `.pdf`, `.md`, and `.txt`.
3. Diagnostic scope:

| Choice | Scope | Run |
|---|---|---|
| A | Format compliance | Author Guidelines checklist only |
| B | Style alignment | Same-journal, same-topic benchmark |
| C | Language style | OA full-text language features |
| D | Section-specific fit | One named section compared with exemplars |
| E | Submission readiness | A plus B; add C or D only if requested |

After the user answers, restate the scope and continue. If all three inputs are already available, proceed without another question.

## Evidence Workflow

### 1. Journal And Guidelines

Resolve journal identity first: title, ISSN, publisher, article type, and target manuscript type when relevant.

For Author Guidelines, prefer sources in this order:

1. Official journal or publisher Author Guidelines page.
2. Reader or extraction services for the official page, if available in the current environment.
3. Search results or third-party aggregators that summarize guidelines.
4. User-pasted guideline text.

If automatic retrieval is needed, read `references/guidelines-fallback-matrix.md` for tested retrieval patterns and failure modes. Use `TAVILY_API_KEY` only when it is already available in the environment; otherwise skip Tavily-specific layers without blocking the task.

If `TAVILY_API_KEY` is absent and the official or reader page returns only a security check, cookie page, empty page, or navigation fragment, do not stop immediately. Try available web search for official mirrors, publisher help pages, or third-party guideline summaries. If those are still weak, ask the user to paste the guideline text.

Extract only requirements supported by the retrieved text:

| Area | Look For |
|---|---|
| Article type | manuscript type, article category, research article |
| Word limits | word, length, page, text limit |
| Abstract | abstract, structured, unstructured, word limit |
| Keywords | keyword count, keyword limit |
| Highlights | highlights, bullet count, character limit |
| Graphical abstract | graphical abstract, required, optional |
| References | reference count, style, format, limits |
| Figures/tables | figure, table, resolution, dpi, limit |
| Statements | data availability, CRediT, ethics, declaration |
| Formatting | line numbers, layout, language variant |

### 2. Draft Features

Parse only the fields needed for the selected scope.

- `.md` and `.txt`: read directly.
- `.docx`: use `python-docx` or an available document parser.
- `.pdf`: use `pymupdf` or an available PDF parser.

Record counts and locations that support the diagnosis: abstract words, keyword count, section headings, figure/table counts, reference count, statement presence, and section-level lengths.

### 3. Same-Journal Benchmarking

Use this only for scope B, C, D, or E.

Select exemplars from the same journal and similar topic. Avoid high-citation but off-topic papers.

- Use journal ISSN plus topic keywords from the draft title, abstract, and keywords.
- Prefer recent research articles, commonly the last 5-6 years unless the field needs a different window.
- Aim for 8-12 usable exemplars. Fewer than 5 means low confidence.
- Use OpenAlex for discovery, Semantic Scholar for abstracts when available, CrossRef/OpenAlex for metadata, and Europe PMC or publisher OA pages for full text when needed.
- For abstracts, try Semantic Scholar first, then reconstruct OpenAlex `abstract_inverted_index` when present, then use a clean CrossRef abstract if available. Exclude no-abstract papers from abstract-length or language statistics and report the usable count.

Summarize benchmarks with median and IQR. Do not rely on mean plus standard deviation for small, skewed samples.

For language-style checks, require enough full text to support the claim. If fewer than 3 OA full texts are usable, label the result as insufficient rather than making a style recommendation.

## Diagnostics

### Format Compliance

Only produce hard compliance findings when the requirement came from real guideline text.

Treat official guideline text and user-pasted official text as eligible for confirmed P0 findings. Treat third-party aggregators and search snippets as fallback evidence only: label them clearly, use them for tentative direction, and mark the item Unknown or "needs official verification" before submission.

| Status | Meaning |
|---|---|
| P0 | Must fix: confirmed journal requirement is unmet |
| Pass | Confirmed compliant |
| Unknown | Requirement could not be verified from available evidence |

### Style Alignment

Use benchmarks for soft conventions, not hard rules.

| Status | Meaning |
|---|---|
| P1 | Strong recommendation: draft is clearly outside the benchmark and evidence confidence is high |
| P2 | Optional optimization: mild deviation or low-confidence evidence |
| Pass | Draft is within the observed journal convention |
| Unknown | Benchmark data is too thin |

## Output Format

Lead with a concise readiness call, then show evidence. Use this structure:

```markdown
## Readiness Call
[Ready / Needs targeted fixes / Not ready], based on [scope].

## Evidence Base
- Guidelines: [official / third-party / pasted / unavailable], source date if visible
- Exemplars: [N] papers, topic filter, data completeness
- Draft: [file/path or pasted text], parsed fields

## P0 - Must Fix
1. [Issue]: current [value]; requirement [value/source] -> revision direction.

## P1 - Recommended
1. [Issue]: current [value]; journal convention [median/IQR, N] -> revision direction.

## P2 - Optional
1. [Issue]: reason and low-risk direction.

## Pass / Unable To Assess
- Pass: [...]
- Unable to assess: [...] because [...]
```

Do not include replacement sentences or rewritten paragraphs. If the user asks for actual wording after the diagnostic, state that this skill's job is complete and switch to an appropriate writing or polishing workflow.

## Common Pitfalls

- Do not infer STOTEN, Wiley, ACS, Springer, or Taylor & Francis requirements from publisher-wide defaults.
- Do not use OpenAlex as an Author Guidelines search engine. Use it for journal identity, article metadata, and same-journal exemplar discovery.
- Do not treat security-check, cookie-consent, or near-empty reader output as guideline text. Wiley and Taylor & Francis pages often fail this way without Tavily or browser access.
- Do not call a generic journal-information or LetPub-style skill unless it is installed and clearly relevant. If unavailable, skip nonessential metadata and continue with official or public sources.
- Do not treat Tavily coverage or third-party aggregator coverage as guaranteed. Use the matrix as a dated test snapshot, not a current fact.
- Do not use CrossRef `page` fields to estimate Elsevier article length; those fields may be article numbers.
- Do not benchmark by ISSN alone. Always add topic keywords.

## References

- `references/guidelines-fallback-matrix.md`: dated retrieval matrix and fallback examples for Author Guidelines.
- `references/pitfalls-and-solutions.md`: observed failure modes and practical corrections from test runs.
