# Learning Notes Hugo Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate all Markdown notes from `learning/` into Hugo content sections using page bundles.

**Architecture:** Treat each first-level `learning/` directory as one Hugo section under `content/`. Convert each Markdown file into a leaf bundle at `content/<section>/<relative-note-path-without-md>/index.md`, copy the source section's image assets into each article bundle so existing `images/...` references keep working, and create `_index.md` files for section list pages.

**Tech Stack:** Hugo, PaperMod theme, POSIX shell/Python 3 for one-time migration, Markdown/TOML front matter.

---

## File Structure

- Modify/create content under `content/`:
  - Create: `content/<section>/_index.md` for each first-level learning section.
  - Create: `content/<section>/<article>/index.md` for each note.
  - Create: `content/<section>/<article>/images/...` by copying image assets from the source section when present.
- Do not modify `learning/`; it remains the source archive.
- Skip `learning/.git`, `.DS_Store`, and non-Markdown/non-image files.
- Verify with Hugo build.

## Tasks

### Task 1: Run the migration script

**Files:**
- Create/Modify: `content/**`
- Source only: `learning/**`

- [ ] **Step 1: Execute the migration script**

Run this command from the repository root:

```bash
python3 - <<'PY'
from pathlib import Path
import re
import shutil
from datetime import datetime, timezone, timedelta

root = Path('.')
source = root / 'learning'
target = root / 'content'
image_exts = {'.png', '.jpg', '.jpeg', '.gif', '.svg', '.webp', '.jpe'}
skip_names = {'.git', '.DS_Store'}
now = datetime.now(timezone(timedelta(hours=8))).replace(microsecond=0).isoformat()

section_titles = {
    'CICD': 'CI/CD',
    'bigData': '大数据',
    'docker': 'Docker',
    'go': 'Go',
    'k8s': 'Kubernetes',
    'linux': 'Linux',
    'log': '日志系统',
    'middleware': '中间件',
    'python': 'Python',
    'shell': 'Shell',
    '前端': '前端',
}

def toml_quote(value: str) -> str:
    return value.replace('\\', '\\\\').replace('"', '\\"')

def has_front_matter(text: str) -> bool:
    stripped = text.lstrip('﻿')
    return stripped.startswith('---\n') or stripped.startswith('+++\n')

def title_from_note(path: Path) -> str:
    return path.stem

def bundle_path(section: Path, md: Path) -> Path:
    rel = md.relative_to(section).with_suffix('')
    return target / section.name / rel / 'index.md'

sections = [p for p in source.iterdir() if p.is_dir() and p.name not in skip_names]
for section in sorted(sections, key=lambda p: p.name.lower()):
    section_target = target / section.name
    section_target.mkdir(parents=True, exist_ok=True)
    section_title = section_titles.get(section.name, section.name)
    section_index = section_target / '_index.md'
    section_index.write_text(
        f'+++\n'
        f'title = "{toml_quote(section_title)}"\n'
        f'date = "{now}"\n'
        f'draft = false\n'
        f'+++\n\n'
        f'{section_title} 相关学习笔记。\n',
        encoding='utf-8'
    )

    section_images = [p for p in section.rglob('*') if p.is_file() and p.suffix.lower() in image_exts]
    notes = [p for p in section.rglob('*.md') if p.is_file()]
    for md in sorted(notes, key=lambda p: str(p).lower()):
        destination = bundle_path(section, md)
        destination.parent.mkdir(parents=True, exist_ok=True)
        text = md.read_text(encoding='utf-8', errors='replace')
        if has_front_matter(text):
            migrated = text
        else:
            migrated = (
                '+++\n'
                f'title = "{toml_quote(title_from_note(md))}"\n'
                f'date = "{now}"\n'
                'draft = false\n'
                '+++\n\n'
                f'{text}'
            )
        destination.write_text(migrated, encoding='utf-8')

        for asset in section_images:
            rel_asset = asset.relative_to(section)
            asset_destination = destination.parent / rel_asset
            asset_destination.parent.mkdir(parents=True, exist_ok=True)
            shutil.copy2(asset, asset_destination)

print(f'Migrated {sum(1 for _ in source.rglob("*.md"))} Markdown files into Hugo page bundles.')
PY
```

Expected: prints a line like `Migrated 46 Markdown files into Hugo page bundles.`

- [ ] **Step 2: Inspect generated structure**

Run:

```bash
find content -maxdepth 3 -type f \( -name '_index.md' -o -name 'index.md' \) | sort | head -80
```

Expected: output includes section indexes such as `content/k8s/_index.md` and bundles such as `content/k8s/docker与containerd/index.md`.

### Task 2: Verify Hugo can render the migrated content

**Files:**
- Read/verify: `content/**`
- Read/verify: `hugo.toml`

- [ ] **Step 1: Run Hugo build including drafts**

Run:

```bash
hugo --buildDrafts
```

Expected: build exits with status 0 and reports rendered pages.

- [ ] **Step 2: Fix any build errors**

If Hugo reports a specific Markdown/front matter/resource error, edit only the generated `content/...` file named in the error. Re-run:

```bash
hugo --buildDrafts
```

Expected: build exits with status 0.

### Task 3: Summarize migration result

**Files:**
- Read/verify: `content/**`

- [ ] **Step 1: Count migrated articles and sections**

Run:

```bash
printf 'sections: '; find content -mindepth 1 -maxdepth 1 -type d | wc -l
printf 'articles: '; find content -type f -name 'index.md' | wc -l
printf 'section indexes: '; find content -type f -name '_index.md' | wc -l
```

Expected: article count matches the number of migrated Markdown source notes, excluding existing sample content if it remains outside the migration sections.

- [ ] **Step 2: Report completion**

Tell the user:

```text
已按页面包方式迁移完成：每个 learning 一级目录变成 Hugo 板块，每篇 Markdown 变成 content/<板块>/<文章>/index.md，并复制图片资源到文章目录下。Hugo build 已通过。
```
