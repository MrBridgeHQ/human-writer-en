# Installation — Human Writer (English) skill

This skill is designed to be installed at the **user level** in Claude Code, so it's available across all your projects without copying it into each repo.

It is the English-language member of the `human-writer` per-language family. Install it alongside the other language skills (`human-writer-fr`, `human-writer-es`, etc.) — they do not conflict; each activates on its own language triggers.

## Prerequisites

- Claude Code installed and working
- Python 3.10+ available on your system (required for the analyzer script)
- `pip` available for installing one mandatory dependency
- A terminal with `unzip` (macOS, Linux) or PowerShell with `Expand-Archive` (Windows)

## Installation

### macOS / Linux

```bash
# 1. Create the user-level skills directory if it doesn't exist
mkdir -p ~/.claude/skills

# 2. Unzip the skill into that directory
unzip human-writer-en.zip -d ~/.claude/skills/

# 3. Install Python dependencies
pip install --user -r ~/.claude/skills/human-writer-en/requirements.txt
# pyyaml is required; httpx is optional (only for --external mode)

# 4. Verify the structure
ls ~/.claude/skills/human-writer-en/
# Expected: SKILL.md  README.md  INSTALL.md  requirements.txt  PROJECT_HISTORY.md  references/  scripts/  tests/

# 5. Make the analyzer script executable (optional, for convenience)
chmod +x ~/.claude/skills/human-writer-en/scripts/analyze.py
```

### Windows (PowerShell)

```powershell
# 1. Create the user-level skills directory if it doesn't exist
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\skills"

# 2. Unzip the skill into that directory
Expand-Archive -Path .\human-writer-en.zip -DestinationPath "$env:USERPROFILE\.claude\skills\"

# 3. Install Python dependencies
pip install --user -r "$env:USERPROFILE\.claude\skills\human-writer-en\requirements.txt"

# 4. Verify the structure
Get-ChildItem "$env:USERPROFILE\.claude\skills\human-writer-en\"
# Expected: SKILL.md  README.md  INSTALL.md  requirements.txt  references/  scripts/
```

## Verification

Open Claude Code and ask:

> "What skills do you have access to?"

You should see `human-writer-en` in the list. If not, check that the SKILL.md file is at the path:

- macOS / Linux: `~/.claude/skills/human-writer-en/SKILL.md`
- Windows: `%USERPROFILE%\.claude\skills\human-writer-en\SKILL.md`

## First test

### Test the skill's auto-activation

In any directory (Claude Code session), ask:

> "Audit this paragraph for AI-detection risk, I'm worried it reads too ChatGPT: [paste a paragraph]"

The skill should activate, route to AUDIT mode, run `scripts/analyze.py --lang en`, and produce a structured report.

### Test the analyzer directly

The analyzer is a standalone CLI. It accepts input from `--input <file>` or stdin, requires `--lang en` and `--type`, and emits either JSON (default) or a human-readable report (`--format human`).

```bash
cd ~/.claude/skills/human-writer-en

# Audit one of the bundled fixtures (a deliberate-AI English sample)
python3 scripts/analyze.py \
  --input tests/fixtures/ai_samples/en_baseline_clean_input_001.md \
  --lang en \
  --type marketing \
  --format human
```

Expected output: a header showing the score, top tells, recommendations. The score on this fixture should land at HIGH_RISK (≥ 50), since this is the deliberate-AI input the skill is calibrated against.

To verify the skill removes tells correctly, run the analyzer against a `refactor_iter_1` output:

```bash
python3 scripts/analyze.py \
  --input tests/fixtures/refactor_iter_1/en_iter1_clean_vinopulse_001.md \
  --lang en \
  --type marketing \
  --format human
```

Expected: score ≤ 5 (LOW_RISK), since the skill fully strips em-dash overuse, suspect vocab, and tricolons.

### Run the test suite

```bash
cd ~/.claude/skills/human-writer-en
python3 -m pytest scripts/test_analyze.py -v
```

Expected: `35 passed, 2 skipped` (the 2 skips are intentional: the parametrized AI-baseline test skips the non-deliberate-AI fixtures, scoring only the `clean_input` variant).

## Optional: external detector setup

The `--external` flag dispatches to one of Copyleaks, GPTZero, or Originality.ai. To use it:

1. Install `httpx` (already in `requirements.txt`).
2. Obtain an API key from the provider and export it:

```bash
export COPYLEAKS_EMAIL="..."       # Copyleaks needs email + key (two-step auth)
export COPYLEAKS_API_KEY="..."     # for --external copyleaks
export GPTZERO_API_KEY="..."       # for --external gptzero
export ORIGINALITY_AI_API_KEY="..." # for --external originality
```

3. Invoke:

```bash
python3 scripts/analyze.py \
  --input draft.md \
  --lang en \
  --type marketing \
  --external copyleaks \
  --format human
```

**Provider verification status:**
- **Copyleaks: live-verified (2026-06-02).** Two-step auth (login with email+key → 48h access token → Bearer on the detector call), response parsed at `summary.ai`. Requires BOTH `COPYLEAKS_EMAIL` and `COPYLEAKS_API_KEY`. Minimum input is ~255 characters (shorter text returns a 400, surfaced as an error, not a crash). `sandbox` mode returns the same shape for free. Free-tier credits are minimal; budget paid credits before looping over `--external copyleaks`.
- **GPTZero / Originality.ai: paused (paid-only API).** Their endpoints and response-parsing JSON paths in `call_gptzero` / `call_originality` remain best-effort reconstructions from public docs, each with an inline `# (verify)` comment.

If `httpx` is not installed, `--external` dispatch raises `ImportError` cleanly without breaking the analyzer's local scoring.

## Updating the skill

To install a newer version, just replace the directory:

```bash
# macOS / Linux
rm -rf ~/.claude/skills/human-writer-en
unzip human-writer-en.zip -d ~/.claude/skills/
pip install --user -r ~/.claude/skills/human-writer-en/requirements.txt
```

Your customizations to `scripts/rules.yaml` (vocab lists, thresholds) will be lost. If you want to preserve them, fork the skill locally before updating.

## Uninstalling

```bash
# macOS / Linux
rm -rf ~/.claude/skills/human-writer-en

# Windows (PowerShell)
Remove-Item -Recurse -Force "$env:USERPROFILE\.claude\skills\human-writer-en"
```

To also uninstall the Python deps (only if you don't use them elsewhere):

```bash
pip uninstall pyyaml httpx
```

## Troubleshooting

**Skill doesn't activate when expected.**
The skill's auto-activation depends on the description in `SKILL.md`'s frontmatter. Triggers include: "humanize", "clean this AI text", "make this less AI", "audit AI detection", "score for Copyleaks". If your phrasing doesn't match, force activation: "Use the `human-writer-en` skill to..."

**Analyzer fails with `ModuleNotFoundError: No module named 'yaml'`.**
`pyyaml` is mandatory. Install with `pip install --user pyyaml`.

**Analyzer fails with `ModuleNotFoundError: No module named 'httpx'` when using `--external`.**
`httpx` is optional and only needed for external detector dispatch. Install with `pip install --user httpx`. Without `--external`, the analyzer runs fully offline.

**External detector returns wrong score or `KeyError` parsing the response.**
The response-parsing logic in `call_copyleaks` / `call_gptzero` / `call_originality` is marked `# (verify)` at the JSON-path lookup. The provider may have changed field names. Run with `--format human` to see the raw response, then edit the JSON path in `scripts/analyze.py` to match.

**Analyzer flags legitimate human English prose as AI (false positive).**
This shouldn't happen at score ≤ 24, but tolerances are content-type-specific. If you find a clear FP:
1. Run `--format human` to see which detector flagged it.
2. Either loosen the threshold in `scripts/rules.yaml` for that detector, or add the flagged construct to a project-specific allowlist.
3. Add the prose as a regression fixture under `tests/fixtures/human_samples/`.

**Tests fail after editing `rules.yaml`.**
The test suite is calibrated to the shipped `rules.yaml`. If you change thresholds or vocab, expect specific tests to fail. Either update the test expectations or revert your change.

**Python 3.10+ syntax errors.**
`analyze.py` uses `from __future__ import annotations` and PEP 604 union syntax (`int | None`). On Python 3.8 or 3.9 this will fail. Upgrade Python or ask for a backport.

## Skill content overview

For the full skill layout, see [`README.md`#whats-inside](README.md#whats-inside).

## Support

This skill is a personal toolkit. If you find the analyzer mis-calibrated against a current detector, or you have to fix a `# (verify)` endpoint in `analyze.py` after a live call, edit the relevant file directly. The skill is meant to track empirical detector behavior, which drifts as providers update their models.

For doctrine improvements (new tells you keep seeing in the wild that aren't in `references/tells-stylistic-en.md`), edit that file directly. It is the authoritative surface; the analyzer is calibrated to follow.
