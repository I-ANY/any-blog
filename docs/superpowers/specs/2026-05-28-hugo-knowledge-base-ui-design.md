# Hugo Knowledge Base UI Design

## Goal

Turn the Hugo site into a professional engineering knowledge base. The home page should help readers quickly understand the available modules and jump into categorized technical notes.

## Chosen Approach

Use PaperMod as the base theme and add lightweight site-level overrides. This preserves PaperMod article rendering, dark mode, table of contents, breadcrumbs, and responsive behavior while adding a custom knowledge-base home page and improved section list pages.

## Information Architecture

### Navigation

Top navigation should include the main technical modules:

- 首页
- Kubernetes
- Docker
- Go
- Linux
- Python
- CI/CD
- 日志
- 中间件

Secondary modules such as Shell, 大数据, 前端 can still appear as cards on the home page even if they are not in the top navigation.

### Home Page

The home page should be structured as:

1. Hero area
   - Title: `ANY的个人博客`
   - Subtitle: notes covering Kubernetes, Docker, Go, Linux, CI/CD, middleware, logging, Python, and frontend topics.
   - Primary action: start reading.
   - Secondary action: view all modules.

2. Module grid
   - One card per first-level content section.
   - Each card shows module title, short description, article count, and a link to the section page.
   - Cards should use concise engineering-focused copy.

3. Recent articles
   - Show recent articles after module cards.
   - Keep this secondary to module navigation.

### Section Pages

Each section page should show:

- Section title.
- Short section description.
- Article cards for all pages in the section.
- Article title, summary if available, and date metadata.

The section page should not add complex search or filtering in this iteration.

### Article Pages

Keep PaperMod's existing single article layout. Do not rewrite article pages unless needed for visual consistency. Existing table of contents, breadcrumbs, and next/previous article behavior should remain intact.

## Visual Direction

The visual style should feel like a technical knowledge base or engineering handbook:

- Clean, high-density layout.
- Card-based module navigation.
- Soft borders and subtle shadows.
- Strong but restrained accent color, leaning blue/cyan/indigo.
- Good Chinese readability.
- Works in both light and dark modes.

## Implementation Boundaries

Prefer project-level overrides instead of editing theme files directly:

- Add or modify `layouts/index.html` for the custom home page.
- Add or modify a section/list layout under `layouts/_default/list.html` or a more specific section layout if needed.
- Add custom CSS under `assets/css/extended/custom.css` or PaperMod's supported extended CSS location.
- Update `hugo.toml` for menus and relevant PaperMod params.
- Add `content/_index.md` if needed for home page metadata.

## Success Criteria

- Home page clearly presents the site as an engineering knowledge base.
- Top navigation exposes the primary modules.
- Every first-level content section is discoverable from the home page.
- Each module page lists its articles clearly.
- Hugo build succeeds.
- No links point to `https://example.org`.
- Existing migrated article content remains readable.
