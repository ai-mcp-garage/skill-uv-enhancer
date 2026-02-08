---
name: skill-uv-enhancer
description: Convert skill scripts to use uv for dependency management. Use when enhancing a skill's Python scripts with PEP 723 inline metadata, converting python invocations to uv run, or ensuring skills have proper dependency isolation. Triggers on "enhance skill", "convert to uv", "add uv to skill", or "fix skill dependencies".
---

# Skill UV Enhancer

Convert an existing skill's Python scripts to use `uv` for dependency management — no virtual environments, no bootstrap scripts, no global installs.

## When This Triggers

- User wants to add proper dependency management to a skill
- User says "enhance", "convert", or "upgrade" a skill's scripts
- User points at a skill directory and wants `uv` support added
- User wants to fix a skill that calls `python` directly

## Prerequisites Check

Before doing anything, verify `uv` is available:

```bash
which uv
```

If missing, tell the user:

```
uv is required but not installed. Install it with:
  curl -LsSf https://astral.sh/uv/install.sh | sh
```

Do NOT proceed until `uv` is confirmed available.

## Enhancement Workflow

### Step 1: Identify the Target Skill

The user provides a path to a skill directory. Verify it has:
- A `SKILL.md` file
- At least one `.py` file (in the root or a `scripts/` subdirectory)

If no Python files exist, tell the user there's nothing to enhance and stop.

### Step 2: Audit Python Scripts

For each `.py` file found, analyze it:

1. **Read the file** and collect all import statements
2. **Classify each import** as stdlib or third-party
3. **Check for existing PEP 723 metadata** (look for `# /// script` block)
4. **Check for a shebang line** at the top

Build a report per script:

| Field | Value |
|-------|-------|
| File | relative path |
| Has PEP 723 metadata | yes/no |
| Stdlib-only | yes/no |
| Third-party imports | list |
| Inferred packages | list (import name -> PyPI package) |

**Common import-to-package mappings** (import name != package name):

| Import | PyPI Package |
|--------|-------------|
| `bs4` | `beautifulsoup4` |
| `cv2` | `opencv-python` |
| `PIL` | `pillow` |
| `sklearn` | `scikit-learn` |
| `yaml` | `pyyaml` |
| `gi` | `pygobject` |
| `attr` | `attrs` |
| `dateutil` | `python-dateutil` |
| `dotenv` | `python-dotenv` |
| `serial` | `pyserial` |
| `usb` | `pyusb` |
| `lxml` | `lxml` |
| `magic` | `python-magic` |

If an import is ambiguous or unknown, flag it for the user to confirm.

**Python stdlib reference** (non-exhaustive, common ones to NOT flag):
`os`, `sys`, `re`, `json`, `math`, `time`, `datetime`, `pathlib`, `subprocess`, `tempfile`, `shutil`, `argparse`, `logging`, `collections`, `itertools`, `functools`, `typing`, `dataclasses`, `enum`, `abc`, `io`, `string`, `random`, `hashlib`, `hmac`, `secrets`, `base64`, `struct`, `copy`, `pprint`, `textwrap`, `urllib`, `http`, `email`, `html`, `xml`, `sqlite3`, `csv`, `configparser`, `tomllib`, `zipfile`, `tarfile`, `gzip`, `bz2`, `lzma`, `socket`, `ssl`, `select`, `threading`, `multiprocessing`, `concurrent`, `asyncio`, `unittest`, `doctest`, `pdb`, `traceback`, `warnings`, `contextlib`, `signal`, `ctypes`, `platform`, `sysconfig`, `glob`, `fnmatch`, `stat`, `fileinput`, `shelve`, `pickle`, `marshal`, `dbm`, `uuid`, `decimal`, `fractions`, `operator`, `array`, `queue`, `heapq`, `bisect`, `weakref`, `types`, `inspect`, `dis`, `code`, `codeop`, `compile`, `ast`, `token`, `tokenize`, `pty`, `tty`, `termios`, `fcntl`, `resource`, `grp`, `pwd`, `posixpath`, `ntpath`, `genericpath`

### Step 3: Present Findings to User

Show the audit results before making changes. Example:

```
Skill: fb-reel-downloader
Scripts found: 1

fb_downloader.py:
  - PEP 723 metadata: NO
  - Stdlib only: YES (all imports are stdlib)
  - Third-party imports: none
  - Action: Skip (no third-party deps needed)

No scripts require uv enhancement. The skill is stdlib-only.
```

Or for a skill with deps:

```
Skill: my-data-tool
Scripts found: 2

scripts/fetch_data.py:
  - PEP 723 metadata: NO
  - Third-party imports: requests, rich
  - Inferred packages: requests, rich
  - Action: ADD inline metadata

scripts/parse_output.py:
  - PEP 723 metadata: NO
  - Stdlib only: YES
  - Action: Skip

SKILL.md references:
  - Line 45: `python scripts/fetch_data.py` → needs `uv run`
  - Line 52: `python scripts/parse_output.py` → safe as-is (stdlib only)
```

**Wait for user confirmation before proceeding.**

### Step 4: Add PEP 723 Inline Metadata

For each script that has third-party dependencies, add the metadata block.

**Placement rules:**
1. After the shebang line (`#!/usr/bin/env python3`) if one exists
2. After the module docstring if one exists
3. Before the first import statement

**Format:**

```python
# /// script
# requires-python = ">=3.11"
# dependencies = [
#     "requests>=2.31",
#     "rich>=13.0",
# ]
# ///
```

**Version pinning strategy:**
- Use `>=` minimum version pins, not exact pins
- Use the latest known major.minor for the minimum (e.g., `requests>=2.31`)
- If unsure about minimum version, use just the package name without a version constraint
- Keep the trailing comma after the last dependency (TOML style)

**If the script already has a PEP 723 block**, check if it's missing any detected dependencies and offer to update it. Do NOT overwrite existing metadata without asking.

### Step 5: Update SKILL.md

Scan the SKILL.md for:

1. **Direct python invocations** — any line containing:
   - `python script_name.py`
   - `python3 script_name.py`
   - `python scripts/script_name.py`
   - `python3 scripts/script_name.py`
   - `python ~/.cursor/skills/*/script.py`
   - Any variation with full paths

2. **Only convert invocations for scripts that have third-party deps.** Stdlib-only scripts can stay as `python` calls — `uv run` still works on them but adds no value.

3. **Replace** `python` / `python3` with `uv run` in those lines.

4. **Add or update the Prerequisites section.** If a Prerequisites section exists, add `uv` to it. If none exists, add one after the first heading.

   Format to add:

   ```markdown
   ## Prerequisites

   - [uv](https://docs.astral.sh/uv/) — Python package runner (handles dependencies automatically)
     ```bash
     curl -LsSf https://astral.sh/uv/install.sh | sh
     ```
   ```

   If there are already other prerequisites listed (like `ffmpeg`), merge `uv` into the existing list rather than creating a duplicate section.

5. **Update Agent Instructions** if a section like "Agent Instructions" or "How to Use" exists and references `python`, update those too.

### Step 6: Shared Requirements File (Optional)

If **3 or more scripts** share the **same set of dependencies**, offer to create a shared `requirements.txt` instead of per-script inline metadata.

In that case:
- Create `scripts/requirements.txt` with the shared deps
- Do NOT add inline metadata to those scripts
- Update SKILL.md invocations to: `uv run --with-requirements <skill-path>/scripts/requirements.txt <skill-path>/scripts/script.py`

For skills with fewer than 3 scripts sharing deps, always prefer inline metadata — it's simpler and self-documenting.

### Step 7: Verify Changes

After making changes:

1. Read back each modified file to confirm edits are correct
2. Check that no `python` or `python3` direct calls remain for scripts with deps
3. Verify PEP 723 blocks are valid TOML
4. Confirm the Prerequisites section exists and mentions `uv`

Present a summary:

```
Enhancement complete:
  ✓ scripts/fetch_data.py — added PEP 723 metadata (requests, rich)
  ✓ SKILL.md — converted 1 python call to uv run
  ✓ SKILL.md — added uv to Prerequisites
  - scripts/parse_output.py — skipped (stdlib only)
```

## Edge Cases

### Script uses only stdlib
Skip it. Don't add unnecessary `uv` overhead. Note it in the report.

### Script already has PEP 723 metadata
Compare detected imports against declared deps. If there's a mismatch, flag it for the user. Don't silently overwrite.

### Script has a `requirements.txt` already
Read it, compare against detected imports, and offer to migrate to inline metadata (or keep it if the user prefers).

### Script imports from other scripts in the same skill
These are local imports, not third-party. Don't try to add them as dependencies. If a script does `from .utils import helper` or `import utils` where `utils.py` is in the same directory, ignore those imports.

### SKILL.md references python in prose (not code blocks)
Only convert invocations inside code blocks or `Agent Instructions` sections. Don't touch descriptive text like "This skill uses Python to..."

### Unknown import
If an import can't be confidently classified as stdlib or mapped to a PyPI package, flag it:

```
⚠ scripts/tool.py imports 'frobulator' — unknown package. Please confirm the PyPI name.
```

## Anti-Patterns

- **Don't add `uv run` to stdlib-only scripts** — it works but adds nothing
- **Don't exact-pin versions** (e.g., `==2.31.0`) — use `>=` minimum pins
- **Don't create a venv** — the whole point is to avoid this
- **Don't modify script logic** — only add metadata blocks and update invocations
- **Don't remove existing prerequisites** — merge `uv` into the existing list
