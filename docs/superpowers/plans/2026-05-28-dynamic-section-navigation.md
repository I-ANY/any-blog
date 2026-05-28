# Dynamic Section Navigation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make new blog sections appear automatically in the homepage module grid and the “博客” navigation dropdown by configuring each section in `content/<section>/_index.md`.

**Architecture:** Use Hugo’s content section model as the single source of truth. `site.Sections.ByWeight` drives ordering; each section’s `_index.md` front matter supplies `title`, `description`, `weight`, and optional `icon`; templates fall back to existing content/title behavior when optional fields are absent.

**Tech Stack:** Hugo templates, TOML front matter, PaperMod theme overrides.

---

## File Structure

- Modify `content/*/_index.md`: add `weight`, `description`, and `icon` to existing top-level section index pages so the current visual order and card copy remain stable.
- Modify `layouts/index.html`: remove hard-coded `$moduleMeta` and `$moduleOrder`; render homepage module cards from `site.Sections.ByWeight`.
- Modify `layouts/_partials/header.html`: make the `blog` top-level menu generate its dropdown children from `site.Sections.ByWeight` instead of manually configured child menu entries.
- Modify `hugo.toml`: remove manually listed blog child menu entries and remove `mainSections`, leaving only top-level navigation and global site params.

## Task 1: Add section metadata to content indexes

**Files:**
- Modify: `content/k8s/_index.md`
- Modify: `content/docker/_index.md`
- Modify: `content/go/_index.md`
- Modify: `content/linux/_index.md`
- Modify: `content/python/_index.md`
- Modify: `content/CICD/_index.md`
- Modify: `content/log/_index.md`
- Modify: `content/middleware/_index.md`
- Modify: `content/shell/_index.md`
- Modify: `content/bigData/_index.md`
- Modify: `content/前端/_index.md`

- [ ] **Step 1: Update each `_index.md` front matter**

For each existing section, keep the existing `title`, `date`, and `draft` values, and add these fields inside the TOML front matter before the closing `+++`.

Use this exact metadata mapping:

```toml
# content/k8s/_index.md
weight = 10
description = "集群安装、网络、监控、排障、备份与迁移"
icon = "K8s"

# content/docker/_index.md
weight = 20
description = "容器运行、镜像、服务部署与运行环境"
icon = "DK"

# content/go/_index.md
weight = 30
description = "Go 语言基础与后端工程实践"
icon = "Go"

# content/linux/_index.md
weight = 40
description = "基础操作、自动化、同步与进程管理"
icon = "Ln"

# content/python/_index.md
weight = 50
description = "Django、Celery、爬虫与常用模块"
icon = "Py"

# content/CICD/_index.md
weight = 60
description = "Jenkins、GitLab、GitOps、Harbor 与交付流水线"
icon = "CI"

# content/log/_index.md
weight = 70
description = "Loki、Elasticsearch、Filebeat、Graylog"
icon = "Log"

# content/middleware/_index.md
weight = 80
description = "Nacos、Dubbo、Sentinel、SkyWalking"
icon = "Mid"

# content/shell/_index.md
weight = 90
description = "Shell 脚本与命令行效率工具"
icon = "Sh"

# content/bigData/_index.md
weight = 100
description = "Hadoop、Kafka 与数据基础设施"
icon = "BD"

# content/前端/_index.md
weight = 110
description = "Vue 与前端工程基础"
icon = "FE"
```

Example resulting file shape:

```toml
+++
title = "Kubernetes"
date = "2026-05-28T00:01:08+08:00"
draft = false
weight = 10
description = "集群安装、网络、监控、排障、备份与迁移"
icon = "K8s"
+++

Kubernetes 相关学习笔记。
```

- [ ] **Step 2: Verify Hugo parses all section metadata**

Run:

```bash
hugo --quiet
```

Expected: command exits with code 0 and no TOML front matter parse errors.

## Task 2: Render homepage modules from Hugo sections

**Files:**
- Modify: `layouts/index.html`

- [ ] **Step 1: Remove hard-coded module maps**

Delete the `$moduleMeta` and `$moduleOrder` definitions at the top of `layouts/index.html`.

The file should start like this:

```go-html-template
{{- define "main" }}
<section class="kb-hero" aria-labelledby="kb-hero-title">
```

- [ ] **Step 2: Replace the module card loop**

Replace the current loop inside `<div class="kb-module-grid">` with this exact loop:

```go-html-template
    {{- range site.Sections.ByWeight }}
      {{- $desc := .Description | default (.Summary | plainify | htmlUnescape | truncate 72) }}
      {{- $icon := .Params.icon | default (substr .Title 0 2) }}
      {{- $count := len .RegularPagesRecursive }}
      <a class="kb-module-card" href="{{ .RelPermalink }}">
        <span class="kb-module-icon">{{ $icon }}</span>
        <span class="kb-module-body">
          <strong>{{ .Title }}</strong>
          <span>{{ $desc }}</span>
        </span>
        <span class="kb-module-count">{{ $count }} 篇</span>
      </a>
    {{- end }}
```

- [ ] **Step 3: Verify homepage builds**

Run:

```bash
hugo --quiet
```

Expected: command exits with code 0. The generated homepage contains module cards for the current top-level content sections in weight order.

## Task 3: Generate the blog dropdown from Hugo sections

**Files:**
- Modify: `layouts/_partials/header.html`

- [ ] **Step 1: Add dynamic blog child detection inside the menu loop**

Inside `{{- range site.Menus.main }}`, after the existing `$is_search` assignment and before the `<li...>` line, add:

```go-html-template
            {{- $isBlogMenu := eq .Identifier "blog" }}
            {{- $hasDynamicChildren := and $isBlogMenu (gt (len site.Sections) 0) }}
```

The surrounding block should become:

```go-html-template
            {{- range site.Menus.main }}
            {{- $menu_item_url := (cond (strings.HasSuffix .URL "/") .URL (printf "%s/" .URL) ) | absLangURL }}
            {{- $page_url:= $currentPage.Permalink | absLangURL }}
            {{- $is_search := eq (site.GetPage .KeyName).Layout `search` }}
            {{- $isBlogMenu := eq .Identifier "blog" }}
            {{- $hasDynamicChildren := and $isBlogMenu (gt (len site.Sections) 0) }}
```

- [ ] **Step 2: Make the dropdown condition include dynamic children**

Change:

```go-html-template
            <li{{ if .HasChildren }} class="has-dropdown"{{ end }}>
                {{- if .HasChildren }}
```

to:

```go-html-template
            <li{{ if or .HasChildren $hasDynamicChildren }} class="has-dropdown"{{ end }}>
                {{- if or .HasChildren $hasDynamicChildren }}
```

- [ ] **Step 3: Render dynamic section children for the blog menu**

Replace the current child loop inside `<ul class="menu-dropdown" aria-label="{{ .Name }} 分类">` with:

```go-html-template
                    {{- if $hasDynamicChildren }}
                    {{- range site.Sections.ByWeight }}
                    <li>
                        <a href="{{ .RelPermalink }}" title="{{ .Title }}">
                            <span>{{ .Title }}</span>
                        </a>
                    </li>
                    {{- end }}
                    {{- else }}
                    {{- range .Children }}
                    <li>
                        <a href="{{ .URL | absLangURL }}" title="{{ .Title | default .Name }}">
                            <span>{{- .Pre }}{{- .Name -}}{{ .Post -}}</span>
                        </a>
                    </li>
                    {{- end }}
                    {{- end }}
```

- [ ] **Step 4: Verify navigation builds**

Run:

```bash
hugo --quiet
```

Expected: command exits with code 0. The generated header contains blog dropdown links for the current top-level content sections.

## Task 4: Remove duplicated section menu configuration

**Files:**
- Modify: `hugo.toml`

- [ ] **Step 1: Remove `mainSections`**

Delete this line from the `[params]` block:

```toml
  mainSections = ['k8s', 'docker', 'go', 'linux', 'python', 'CICD', 'log', 'middleware', 'shell', 'bigData', '前端']
```

- [ ] **Step 2: Remove child `menu.main` entries under blog**

Keep only these top-level menu entries:

```toml
[menu]
  [[menu.main]]
    name = '首页'
    url = '/'
    weight = 1
  [[menu.main]]
    identifier = 'blog'
    name = '博客'
    weight = 10
```

Delete every `[[menu.main]]` block that has `parent = 'blog'`, including Kubernetes, Docker, Go, Linux, Python, CI/CD, 日志, and 中间件.

- [ ] **Step 3: Verify config builds without duplicated menu entries**

Run:

```bash
hugo --quiet
```

Expected: command exits with code 0. The generated header still has 首页 and 博客, and 博客 dropdown is populated by content sections rather than TOML child menu entries.

## Task 5: Manual verification for adding a new section

**Files:**
- Create then remove: `content/test-section/_index.md`

- [ ] **Step 1: Create a temporary section index**

Create `content/test-section/_index.md` with:

```toml
+++
title = "测试板块"
date = "2026-05-28T00:00:00+08:00"
draft = false
weight = 999
description = "用于验证新增板块是否自动出现在导航和首页"
icon = "TS"
+++
```

- [ ] **Step 2: Build with the temporary section**

Run:

```bash
hugo --quiet
```

Expected: command exits with code 0.

- [ ] **Step 3: Verify generated homepage contains the temporary card**

Run:

```bash
grep -n "测试板块" public/index.html
```

Expected: output includes a line containing `测试板块`.

- [ ] **Step 4: Verify generated header contains the temporary dropdown item**

Run:

```bash
grep -R -n "测试板块" public/index.html | head
```

Expected: output includes at least one header dropdown occurrence and one homepage module card occurrence.

- [ ] **Step 5: Remove the temporary section**

Remove `content/test-section/_index.md` and the empty `content/test-section/` directory.

- [ ] **Step 6: Rebuild after cleanup**

Run:

```bash
hugo --quiet
```

Expected: command exits with code 0, and `grep -R -n "测试板块" public/index.html` returns no matches.
