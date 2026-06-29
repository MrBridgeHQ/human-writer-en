# Human Writer — English (en) — a Claude skill

A Claude Code skill that produces **English** prose reading as human-authored, sanitizes English AI drafts to remove detector tells, and scores any English draft for AI-detection risk before publication.

This is the **English member** of the `human-writer` per-language family. It was extracted from the bilingual `human-writer` master (the master held authentic EN + FR doctrine, EN fixtures, and the EN test suite); this skill carries the English half. Its French counterpart is [`human-writer-fr`](../human-writer-fr/). The two install side by side and do not conflict: each activates on its own language triggers. The family has seven autonomous, mono-lingual skills (`en`, `fr`, `es`, `pt`, `de`, `ar`, `hi`) built on the same architecture.

## Why this skill exists

Modern Sonnet / Opus / GPT-class output is fluent enough to ship, but it carries fingerprints that commercial detectors (Copyleaks, GPTZero, Originality.ai) latch onto. Statistical fingerprints: low burstiness, narrow type-token ratio. Stylistic ones: em-dash overuse, the same forty or fifty verbs repeated across drafts ("delve", "leverage", "seamless", "robust"), formulaic frames of the "It's not just X, it's Y" shape. Structural ones: header pyramids, tricolons every second paragraph, bullets where every item opens with the same verb.

Most teams discover this the hard way. A post drops, a client runs it through a detector, the score comes back at 70%+, and the draft is rejected. The fix is not "rewrite from scratch": it is a disciplined sweep of the specific tells the detectors weight. This skill encodes that doctrine plus a deterministic analyzer so any Claude Code session can produce, clean, or audit English prose to a target score before delivery.

## What it can do

**Three modes:**
- **Write:** produce new English content already engineered to score LOW_RISK.
- **Clean:** rewrite an existing English AI draft to strip tells, preserving meaning.
- **Audit:** score an English draft, surface the top offending tells, recommend fixes without rewriting it.

**One language, fully specialized:**
- **English (en):** ~200-entry suspect vocabulary (Tier 1 + Tier 2), a 54-pattern English AI-construction regex bank, an English tricolon detector ("X, Y, and Z"), and English-specific em-dash discipline. English's only typographic tell is the literal em-dash "—" (U+2014), which the analyzer already counts; there is no separate typography detector.

**Four content-types** (each with its own adapter): marketing long-form, short-comms, technical, editorial-SEO.

**Optional external integration:** live scoring via Copyleaks, GPTZero, or Originality.ai (`--external <provider>`). Lazy `httpx` import, so the analyzer works fully offline without the dependency.

## What's inside

```
human-writer-en/
├── SKILL.md                          # Orchestration hub: routing + master checklist + anti-patterns (en)
├── README.md                         # This file
├── INSTALL.md                        # Installation (path-adjusted)
├── requirements.txt                  # pyyaml (required) + httpx (optional)
├── PROJECT_HISTORY.md                # Extraction journal + provenance note (TRACKED)
├── references/
│   ├── tells-stylistic-en.md         # ⭐ CORE: ~200 EN suspect vocab + AI constructions + em-dash discipline
│   ├── tells-statistical.md          # Burstiness, TTR, comma density doctrine
│   ├── tells-structural.md           # Bullet parallelism, header pyramids, tricolons, emoji
│   ├── humanization-techniques.md    # 10 core techniques with worked examples
│   ├── adapter-marketing.md          # Long-form tolerances + content-type checklist
│   ├── adapter-short-comms.md        # Email / LinkedIn / Slack tighter discipline
│   ├── adapter-technical.md          # RFC / engineering blog / design doc tolerances
│   ├── adapter-editorial-seo.md      # Editorial long-form + SEO E-E-A-T markers
│   ├── external-detectors.md         # Copyleaks / GPTZero / Originality doctrine + response shapes
│   ├── checklists.md                 # Pre-publish checklists per content-type
│   └── _self-audit.md                # Self-audit checklist for the skill's own outputs
├── scripts/
│   ├── rules.yaml                    # ⭐ en: vocab + constructions + thresholds + content_type_weights + 4-band verdict_bands
│   ├── analyze.py                    # ⭐ en-only deterministic scorer (0-100) + external dispatch
│   └── test_analyze.py               # ⭐ EN pytest suite
└── tests/fixtures/
    ├── ai_samples/                   # baseline AI-generated English fixtures
    ├── human_samples/                # genuine human English prose for FP-rate calibration
    └── refactor_iter_1/              # skill-applied (cleaned) English outputs
```

## Installation

See `INSTALL.md`. TL;DR for macOS/Linux:

```bash
mkdir -p ~/.claude/skills
unzip human-writer-en.zip -d ~/.claude/skills/
pip install --user -r ~/.claude/skills/human-writer-en/requirements.txt
```

## How to invoke

Once installed, the skill auto-activates on English prose requests. Example prompts:

- "Write a 600-word blog post about wine futures pricing, make it read human, not AI"
- "Clean this AI draft to bring its Copyleaks score under 25"
- "Audit this email for AI tells before I send it to the client"
- "Score this article for AI-detection risk and tell me what to fix first"
- "Rewrite the overview paragraph of this README so it doesn't pattern-match as AI"

If auto-activation misses, force it: "Use the `human-writer-en` skill to..."

## The analyzer

`scripts/analyze.py` is a deterministic scorer that runs offline. It loads `scripts/rules.yaml` (English vocab lists, regex patterns, thresholds) and emits a 0-100 score plus a list of flagged tells.

```bash
# Audit mode (default, JSON output for tooling)
python3 scripts/analyze.py --input draft.md --lang en --type marketing

# Human-readable report
python3 scripts/analyze.py --input draft.md --lang en --type editorial-seo --format human

# Live external detector (requires API key in env: COPYLEAKS_API_KEY, etc.)
python3 scripts/analyze.py --input draft.md --lang en --type marketing --external copyleaks

# Pipe via stdin
cat draft.md | python3 scripts/analyze.py --lang en --type technical --format human
```

Required flags: `--lang en`, `--type` (`marketing` / `short-comms` / `technical` / `editorial-seo`). `--input` is optional; stdin otherwise.

**Score bands** (4-band YAML, canonical):
- `0-24` **LOW_RISK:** ship it.
- `25-49` **MEDIUM_RISK:** apply the top 3 recommendations, re-score.
- `50-74` **HIGH_RISK:** in WRITE mode restart; in CLEAN mode apply a stronger rewrite.
- `75-100` **CRITICAL:** major rewrite.

**Detectors implemented:** em-dash density, sentence-length stdev (burstiness), TTR (lexical diversity), English suspect-vocabulary, English AI-construction regex bank, English tricolon ("X, Y, and Z"), bullet parallelism, and header pyramid.

**Prose-only scoring.** The analyzer strips fenced code blocks, markdown data tables, and opt-in ignore regions before scoring. Wrap any intentionally AI-flavored citation so it doesn't count against you:

```
<!-- human-writer:ignore-start (quoted bad example) -->
In today's fast-paced world, we delve into seamless, robust solutions.
<!-- human-writer:ignore-end -->
```

This is what lets a how-to-spot-AI article quote bad examples without flagging itself. The start marker accepts a trailing annotation (the reason).

## What's NOT inside

- **French content:** use `human-writer-fr`. Spanish / Portuguese / German / Arabic / Hindi: use the matching satellite.
- **Document structure** (which sections, which schemas, which headings): use a dedicated structure/content tool. This skill is the stylistic filter applied on top of whatever produces the structure.
- **Technical SEO audit of a web project:** use a dedicated SEO tool. This skill handles content style only.

This skill is a stylistic filter invoked on top of structure-producing tools, never as a replacement.

## Part of the mr-bridge.com toolkit

This skill is part of the [mr-bridge.com](https://mr-bridge.com) toolkit for scraping, data, and content automation. Related resources:

- [mr-bridge.com](https://mr-bridge.com) — home
- [Scrapers](https://mr-bridge.com/scrapers) — the Apify Actor portfolio
- [MCP servers](https://mr-bridge.com/mcp-servers) — Model Context Protocol servers
- [AI workflows](https://mr-bridge.com/ai-workflows) — agents and automation
- [Studies](https://mr-bridge.com/studies) — data studies and one-pagers
- [Articles](https://mr-bridge.com/articles) — write-ups and guides
- [Solutions](https://mr-bridge.com/solutions) — end-to-end solutions

## License

Personal use. Customize freely. No warranty. The external-detector endpoints in `analyze.py` carry `# (verify)` markers; Copyleaks was live-verified (2026-06-02), GPTZero / Originality.ai remain best-effort reconstructions from public docs. They are language-agnostic POSTs and were not re-verified for English specifically.
