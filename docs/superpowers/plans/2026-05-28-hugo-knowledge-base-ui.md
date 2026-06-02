# Hugo Knowledge Base UI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a professional knowledge-base UI for the Hugo/PaperMod site with top navigation, module cards, categorized article lists, and polished responsive styling.

**Architecture:** Keep PaperMod as the base theme and add project-level overrides only. Implement a custom home page at `layouts/index.html`, a custom section list page at `layouts/_default/list.html`, a PaperMod extended stylesheet at `assets/css/extended/custom.css`, and menu/site metadata in `hugo.toml`.

**Tech Stack:** Hugo templates, PaperMod theme conventions, TOML configuration, CSS custom properties/responsive CSS.

---

## File Structure

- Create: `layouts/index.html` — custom knowledge-base home page with hero, module cards, and recent articles.
- Create: `layouts/_default/list.html` — custom section list page for modules while preserving generic list behavior.
- Create: `assets/css/extended/custom.css` — visual styling for hero, cards, navigation refinements, section lists, and dark mode.
- Modify: `hugo.toml` — site title/description, PaperMod params, and top navigation menu.
- Create: `content/_index.md` — home page content metadata.

## Tasks

### Task 1: Configure Hugo navigation and home metadata

**Files:**
- Modify: `hugo.toml`
- Create: `content/_index.md`

- [ ] **Step 1: Update `hugo.toml`**

Replace the current file content with:

```toml
baseURL = '/'
locale = 'zh-cn'
languageCode = 'zh-cn'
title = 'ANY的个人博客'
theme = 'PaperMod'

[params]
  description = 'Kubernetes、Docker、Go、Linux、CI/CD、中间件、日志系统等工程学习笔记。'
  author = 'any'
  defaultTheme = 'auto'
  ShowReadingTime = true
  ShowBreadCrumbs = true
  ShowPostNavLinks = true
  ShowToc = true
  TocOpen = true
  mainSections = ['k8s', 'docker', 'go', 'linux', 'python', 'CICD', 'log', 'middleware', 'shell', 'bigData', '前端']

[menu]
  [[menu.main]]
    name = '首页'
    url = '/'
    weight = 1
  [[menu.main]]
    name = 'Kubernetes'
    url = '/k8s/'
    weight = 10
  [[menu.main]]
    name = 'Docker'
    url = '/docker/'
    weight = 20
  [[menu.main]]
    name = 'Go'
    url = '/go/'
    weight = 30
  [[menu.main]]
    name = 'Linux'
    url = '/linux/'
    weight = 40
  [[menu.main]]
    name = 'Python'
    url = '/python/'
    weight = 50
  [[menu.main]]
    name = 'CI/CD'
    url = '/cicd/'
    weight = 60
  [[menu.main]]
    name = '日志'
    url = '/log/'
    weight = 70
  [[menu.main]]
    name = '中间件'
    url = '/middleware/'
    weight = 80
```

- [ ] **Step 2: Create `content/_index.md`**

Create the file with:

```markdown
+++
title = 'ANY的个人博客'
description = '按 Kubernetes、Docker、Go、Linux、CI/CD、中间件、日志系统等模块整理的工程学习笔记。'
draft = false
+++
```

- [ ] **Step 3: Verify menu config parses**

Run:

```bash
hugo --buildDrafts
```

Expected: exit 0. Raw HTML warnings from migrated notes are acceptable.

### Task 2: Implement the knowledge-base home page

**Files:**
- Create: `layouts/index.html`

- [ ] **Step 1: Create `layouts/index.html`**

Create the file with:

```go-html-template
{{- define "main" }}
{{- $moduleMeta := dict
  "k8s" (dict "title" "Kubernetes" "desc" "集群安装、网络、监控、排障、备份与迁移" "icon" "K8s")
  "docker" (dict "title" "Docker" "desc" "容器运行、镜像、服务部署与运行环境" "icon" "DK")
  "go" (dict "title" "Go" "desc" "Go 语言基础与后端工程实践" "icon" "Go")
  "linux" (dict "title" "Linux" "desc" "基础操作、自动化、同步与进程管理" "icon" "Ln")
  "python" (dict "title" "Python" "desc" "Django、Celery、爬虫与常用模块" "icon" "Py")
  "CICD" (dict "title" "CI/CD" "desc" "Jenkins、GitLab、GitOps、Harbor 与交付流水线" "icon" "CI")
  "log" (dict "title" "日志系统" "desc" "Loki、Elasticsearch、Filebeat、Graylog" "icon" "Log")
  "middleware" (dict "title" "中间件" "desc" "Nacos、Dubbo、Sentinel、SkyWalking" "icon" "Mid")
  "shell" (dict "title" "Shell" "desc" "Shell 脚本与命令行效率工具" "icon" "Sh")
  "bigData" (dict "title" "大数据" "desc" "Hadoop、Kafka 与数据基础设施" "icon" "BD")
  "前端" (dict "title" "前端" "desc" "Vue 与前端工程基础" "icon" "FE")
}}
{{- $moduleOrder := slice "k8s" "docker" "go" "linux" "python" "CICD" "log" "middleware" "shell" "bigData" "前端" }}

<section class="kb-hero" aria-labelledby="kb-hero-title">
  <div class="kb-hero-content">
    <p class="kb-eyebrow">Engineering Knowledge Base</p>
    <h1 id="kb-hero-title">ANY的个人博客</h1>
    <p class="kb-hero-copy">按模块沉淀 Kubernetes、Docker、Go、Linux、CI/CD、中间件、日志系统等工程学习笔记，让排障、部署和复盘都有清晰入口。</p>
    <div class="kb-hero-actions">
      <a class="kb-button kb-button-primary" href="#modules">开始阅读</a>
      <a class="kb-button kb-button-secondary" href="#recent">最近更新</a>
    </div>
  </div>
  <div class="kb-hero-panel" aria-label="知识库统计">
    <span class="kb-panel-label">当前收录</span>
    <strong>{{ len site.RegularPages }}</strong>
    <span>篇工程笔记</span>
  </div>
</section>

<section class="kb-section" id="modules" aria-labelledby="modules-title">
  <div class="kb-section-heading">
    <p class="kb-eyebrow">Modules</p>
  </div>
  <div class="kb-module-grid">
    {{- range $moduleOrder }}
      {{- $section := site.GetPage "section" . }}
      {{- if $section }}
        {{- $meta := index $moduleMeta . }}
        {{- $count := len $section.RegularPagesRecursive }}
        <a class="kb-module-card" href="{{ $section.RelPermalink }}">
          <span class="kb-module-icon">{{ index $meta "icon" }}</span>
          <span class="kb-module-body">
            <strong>{{ index $meta "title" }}</strong>
            <span>{{ index $meta "desc" }}</span>
          </span>
          <span class="kb-module-count">{{ $count }} 篇</span>
        </a>
      {{- end }}
    {{- end }}
  </div>
</section>

<section class="kb-section" id="recent" aria-labelledby="recent-title">
  <div class="kb-section-heading">
    <p class="kb-eyebrow">Recently Updated</p>
    <h2 id="recent-title">最近文章</h2>
    <p>从最近整理的内容开始阅读，也可以回到上方按模块查找。</p>
  </div>
  <div class="kb-recent-list">
    {{- range first 10 site.RegularPages.ByDate.Reverse }}
      <article class="kb-article-card">
        <a href="{{ .RelPermalink }}">
          <h3>{{ .Title }}</h3>
          <p>{{ .Summary | plainify | htmlUnescape | truncate 120 }}</p>
          <span>{{ .Date.Format "2006-01-02" }}</span>
        </a>
      </article>
    {{- end }}
  </div>
</section>
{{- end }}
```

- [ ] **Step 2: Verify home page builds**

Run:

```bash
hugo --buildDrafts
```

Expected: exit 0.

### Task 3: Implement categorized section list pages

**Files:**
- Create: `layouts/_default/list.html`

- [ ] **Step 1: Create `layouts/_default/list.html`**

Create the file with:

```go-html-template
{{- define "main" }}
{{- if .IsHome }}
  {{- partial "index.html" . }}
{{- else }}
  <header class="kb-list-header">
    {{- partial "breadcrumbs.html" . }}
    <p class="kb-eyebrow">Module</p>
    <h1>{{ .Title }}</h1>
    {{- if .Description }}
      <p>{{ .Description | markdownify }}</p>
    {{- else if .Content }}
      <div class="kb-list-description">{{ .Content }}</div>
    {{- end }}
  </header>

  {{- $pages := union .RegularPages .Sections }}
  {{- $paginator := .Paginate $pages }}
  <section class="kb-list-grid" aria-label="{{ .Title }} 文章列表">
    {{- range $paginator.Pages }}
      <article class="kb-list-card">
        <a href="{{ .RelPermalink }}">
          <div>
            <h2>{{ .Title }}</h2>
            {{- if (ne (.Param "hideSummary") true) }}
              <p>{{ .Summary | plainify | htmlUnescape | truncate 140 }}</p>
            {{- end }}
          </div>
          <footer>
            <span>{{ .Date.Format "2006-01-02" }}</span>
            <span>阅读全文 →</span>
          </footer>
        </a>
      </article>
    {{- end }}
  </section>

  {{- if gt $paginator.TotalPages 1 }}
    <footer class="page-footer">
      <nav class="pagination">
        {{- if $paginator.HasPrev }}
          <a class="prev" href="{{ $paginator.Prev.URL | relURL }}">« 上一页</a>
        {{- end }}
        {{- if $paginator.HasNext }}
          <a class="next" href="{{ $paginator.Next.URL | relURL }}">下一页 »</a>
        {{- end }}
      </nav>
    </footer>
  {{- end }}
{{- end }}
{{- end }}
```

- [ ] **Step 2: Verify section pages build**

Run:

```bash
hugo --buildDrafts
```

Expected: exit 0.

### Task 4: Add polished knowledge-base styling

**Files:**
- Create: `assets/css/extended/custom.css`

- [ ] **Step 1: Create `assets/css/extended/custom.css`**

Create the file with:

```css
:root {
  --kb-accent: #2563eb;
  --kb-accent-2: #06b6d4;
  --kb-card: rgba(255, 255, 255, 0.78);
  --kb-card-border: rgba(37, 99, 235, 0.14);
  --kb-soft-bg: linear-gradient(135deg, rgba(37, 99, 235, 0.10), rgba(6, 182, 212, 0.08));
  --kb-shadow: 0 18px 45px rgba(15, 23, 42, 0.08);
}

.dark {
  --kb-card: rgba(17, 24, 39, 0.72);
  --kb-card-border: rgba(148, 163, 184, 0.20);
  --kb-soft-bg: linear-gradient(135deg, rgba(37, 99, 235, 0.24), rgba(6, 182, 212, 0.14));
  --kb-shadow: 0 18px 45px rgba(0, 0, 0, 0.22);
}

.logo a {
  font-weight: 800;
  letter-spacing: -0.03em;
}

#menu a {
  border-radius: 999px;
  padding: 6px 12px;
  transition: background-color 0.2s ease, color 0.2s ease;
}

#menu a:hover {
  background: var(--kb-soft-bg);
  color: var(--kb-accent);
}

.kb-hero {
  display: grid;
  grid-template-columns: minmax(0, 1fr) 280px;
  gap: 28px;
  align-items: stretch;
  margin: 32px 0 42px;
  padding: 34px;
  border: 1px solid var(--kb-card-border);
  border-radius: 28px;
  background: var(--kb-soft-bg);
  box-shadow: var(--kb-shadow);
}

.kb-eyebrow {
  margin: 0 0 10px;
  color: var(--kb-accent);
  font-size: 0.78rem;
  font-weight: 800;
  letter-spacing: 0.12em;
  text-transform: uppercase;
}

.kb-hero h1 {
  margin: 0;
  font-size: clamp(2.1rem, 5vw, 4.2rem);
  letter-spacing: -0.06em;
  line-height: 1.05;
}

.kb-hero-copy {
  max-width: 760px;
  margin: 18px 0 0;
  color: var(--secondary);
  font-size: 1.05rem;
  line-height: 1.9;
}

.kb-hero-actions {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
  margin-top: 26px;
}

.kb-button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-height: 42px;
  padding: 0 18px;
  border-radius: 999px;
  font-weight: 700;
}

.kb-button-primary {
  background: var(--kb-accent);
  color: #fff;
}

.kb-button-secondary {
  border: 1px solid var(--kb-card-border);
  background: var(--kb-card);
}

.kb-hero-panel {
  display: flex;
  min-height: 190px;
  flex-direction: column;
  justify-content: center;
  padding: 26px;
  border: 1px solid var(--kb-card-border);
  border-radius: 24px;
  background: var(--kb-card);
}

.kb-hero-panel strong {
  font-size: 4rem;
  letter-spacing: -0.08em;
  line-height: 1;
}

.kb-panel-label,
.kb-hero-panel span:last-child {
  color: var(--secondary);
  font-weight: 700;
}

.kb-section {
  margin: 42px 0;
}

.kb-section-heading {
  max-width: 720px;
  margin-bottom: 20px;
}

.kb-section-heading h2 {
  margin: 0;
  font-size: clamp(1.6rem, 3vw, 2.35rem);
  letter-spacing: -0.04em;
}

.kb-section-heading p:last-child {
  margin-top: 8px;
  color: var(--secondary);
}

.kb-module-grid {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 16px;
}

.kb-module-card,
.kb-article-card a,
.kb-list-card a {
  display: flex;
  height: 100%;
  border: 1px solid var(--kb-card-border);
  border-radius: 20px;
  background: var(--kb-card);
  box-shadow: 0 10px 24px rgba(15, 23, 42, 0.04);
  transition: transform 0.2s ease, border-color 0.2s ease, box-shadow 0.2s ease;
}

.kb-module-card {
  position: relative;
  gap: 14px;
  min-height: 138px;
  padding: 18px;
}

.kb-module-card:hover,
.kb-article-card a:hover,
.kb-list-card a:hover {
  transform: translateY(-3px);
  border-color: rgba(37, 99, 235, 0.42);
  box-shadow: var(--kb-shadow);
}

.kb-module-icon {
  display: inline-flex;
  width: 46px;
  height: 46px;
  flex: 0 0 auto;
  align-items: center;
  justify-content: center;
  border-radius: 14px;
  background: var(--kb-soft-bg);
  color: var(--kb-accent);
  font-size: 0.82rem;
  font-weight: 900;
}

.kb-module-body {
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding-right: 54px;
}

.kb-module-body strong {
  font-size: 1.08rem;
}

.kb-module-body span {
  color: var(--secondary);
  font-size: 0.92rem;
  line-height: 1.65;
}

.kb-module-count {
  position: absolute;
  top: 18px;
  right: 18px;
  color: var(--secondary);
  font-size: 0.82rem;
  font-weight: 800;
}

.kb-recent-list,
.kb-list-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 16px;
}

.kb-article-card a,
.kb-list-card a {
  flex-direction: column;
  justify-content: space-between;
  padding: 20px;
}

.kb-article-card h3,
.kb-list-card h2 {
  margin: 0;
  font-size: 1.08rem;
  line-height: 1.45;
}

.kb-article-card p,
.kb-list-card p,
.kb-list-header p,
.kb-list-description {
  color: var(--secondary);
  line-height: 1.75;
}

.kb-article-card span,
.kb-list-card footer {
  color: var(--secondary);
  font-size: 0.84rem;
  font-weight: 700;
}

.kb-list-header {
  margin: 28px 0 28px;
  padding: 28px;
  border: 1px solid var(--kb-card-border);
  border-radius: 24px;
  background: var(--kb-soft-bg);
}

.kb-list-header h1 {
  margin: 0;
  font-size: clamp(2rem, 4vw, 3.4rem);
  letter-spacing: -0.06em;
}

.kb-list-card footer {
  display: flex;
  justify-content: space-between;
  gap: 14px;
  margin-top: 18px;
}

@media (max-width: 900px) {
  .kb-hero,
  .kb-module-grid,
  .kb-recent-list,
  .kb-list-grid {
    grid-template-columns: 1fr;
  }

  .kb-hero {
    padding: 24px;
  }
}

@media (max-width: 640px) {
  .kb-hero-actions {
    flex-direction: column;
  }

  .kb-button {
    width: 100%;
  }

  .kb-module-card {
    min-height: auto;
  }
}
```

- [ ] **Step 2: Verify styles are bundled by PaperMod**

Run:

```bash
hugo --buildDrafts
```

Expected: exit 0 and generated CSS includes `kb-hero` styles.

### Task 5: Verify rendered output and links

**Files:**
- Verify: `public/**`

- [ ] **Step 1: Verify build and generated markup**

Run:

```bash
hugo --buildDrafts && grep -R "kb-hero" -n public/index.html && grep -R "https://example.org" -n public 2>/dev/null | head -20
```

Expected: Hugo build exits 0, `kb-hero` appears in `public/index.html`, and the final grep prints no `https://example.org` results.

- [ ] **Step 2: Start the Hugo server for manual UI verification**

Run:

```bash
hugo server --buildDrafts --bind 127.0.0.1 --port 1313
```

Expected: server starts and prints a local URL such as `http://127.0.0.1:1313/`.

- [ ] **Step 3: Manually inspect in browser**

Open `http://127.0.0.1:1313/` and verify:

```text
- 首页显示 Hero、模块卡片、最近文章。
- 顶部导航包含 首页、Kubernetes、Docker、Go、Linux、Python、CI/CD、日志、中间件。
- 点击 Kubernetes 进入 /k8s/，文章按卡片展示。
- 点击一篇文章能进入文章详情，不跳转到 Example Domain。
- 页面在窄屏宽度下模块卡片单列展示。
```
