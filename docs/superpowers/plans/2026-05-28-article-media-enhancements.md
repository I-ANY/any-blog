# Article Media Enhancements Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enable code-block copy buttons site-wide and add click-to-zoom image preview with mouse-wheel scaling for article content.

**Architecture:** Use PaperMod's existing `ShowCodeCopyButtons` footer script for code copying by enabling the site parameter in Hugo config. Add a site-owned footer extension partial for `.post-content img` lightbox behavior, and keep all visual styling in the existing extended custom CSS.

**Tech Stack:** Hugo, PaperMod theme, vanilla JavaScript, CSS.

---

## File Structure

- Modify `hugo.toml`: add `ShowCodeCopyButtons = true` under `[params]` so PaperMod's existing copy button script runs globally for page content.
- Create `layouts/partials/extend_footer.html`: PaperMod includes this extension partial from its footer; it will initialize image zoom for `.post-content img` without editing theme source.
- Modify `assets/css/extended/custom.css`: add image cursor styling and modal/lightbox styles matching the existing blue card aesthetic.

## Task 1: Enable PaperMod code copy buttons globally

**Files:**
- Modify: `hugo.toml`

- [ ] **Step 1: Add the global parameter**

In `hugo.toml`, add this line inside the existing `[params]` block, near the other `Show...` settings:

```toml
  ShowCodeCopyButtons = true
```

The `[params]` block should include:

```toml
[params]
  description = 'Kubernetes、Docker、Go、Linux、CI/CD、中间件、日志系统等工程学习笔记。'
  author = 'ANY'
  defaultTheme = 'auto'
  ShowReadingTime = true
  ShowBreadCrumbs = true
  ShowPostNavLinks = true
  ShowToc = true
  TocOpen = true
  ShowCodeCopyButtons = true
  mainSections = ['k8s', 'docker', 'go', 'linux', 'python', 'CICD', 'log', 'middleware', 'shell', 'bigData', '前端']
```

- [ ] **Step 2: Verify Hugo config parses**

Run:

```bash
hugo --quiet
```

Expected: command exits with code 0 and builds the site without TOML parse errors.

## Task 2: Add article image zoom behavior

**Files:**
- Create: `layouts/partials/extend_footer.html`

- [ ] **Step 1: Create the footer extension partial**

Create `layouts/partials/extend_footer.html` with exactly this content:

```html
{{- if eq .Kind "page" }}
<script>
  (() => {
    const images = Array.from(document.querySelectorAll('.post-content img'));
    if (!images.length) return;

    let scale = 1;
    let activeImage = null;

    const overlay = document.createElement('div');
    overlay.className = 'kb-image-lightbox';
    overlay.innerHTML = '<button class="kb-image-lightbox-close" type="button" aria-label="关闭图片预览">×</button><img alt="">';
    document.body.appendChild(overlay);

    const preview = overlay.querySelector('img');
    const closeButton = overlay.querySelector('button');

    function applyScale() {
      preview.style.transform = `scale(${scale})`;
    }

    function closePreview() {
      overlay.classList.remove('is-active');
      document.body.classList.remove('kb-lightbox-open');
      preview.removeAttribute('src');
      preview.alt = '';
      activeImage = null;
      scale = 1;
      applyScale();
    }

    function openPreview(image) {
      activeImage = image;
      scale = 1;
      preview.src = image.currentSrc || image.src;
      preview.alt = image.alt || '';
      applyScale();
      overlay.classList.add('is-active');
      document.body.classList.add('kb-lightbox-open');
    }

    images.forEach((image) => {
      image.addEventListener('click', () => openPreview(image));
    });

    overlay.addEventListener('click', (event) => {
      if (event.target === overlay) closePreview();
    });

    closeButton.addEventListener('click', closePreview);

    overlay.addEventListener('wheel', (event) => {
      if (!activeImage) return;
      event.preventDefault();
      const direction = event.deltaY > 0 ? -1 : 1;
      scale = Math.min(4, Math.max(0.5, scale + direction * 0.12));
      applyScale();
    }, { passive: false });

    document.addEventListener('keydown', (event) => {
      if (event.key === 'Escape' && activeImage) closePreview();
    });
  })();
</script>
{{- end }}
```

- [ ] **Step 2: Verify the partial is picked up by the theme**

Run:

```bash
hugo --quiet
```

Expected: command exits with code 0. The generated article pages include the script because PaperMod's footer calls `partial "extend_footer.html" .`.

## Task 3: Style clickable article images and the lightbox

**Files:**
- Modify: `assets/css/extended/custom.css`

- [ ] **Step 1: Add image and lightbox styles**

Add this CSS after the existing `.post-content { font-size: 16px; }` block:

```css
.post-content img {
  cursor: zoom-in;
}

.kb-lightbox-open {
  overflow: hidden;
}

.kb-image-lightbox {
  position: fixed;
  inset: 0;
  z-index: 9999;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 32px;
  background: rgba(15, 23, 42, 0.82);
  opacity: 0;
  visibility: hidden;
  transition: opacity 0.18s ease, visibility 0.18s ease;
  backdrop-filter: blur(14px);
}

.kb-image-lightbox.is-active {
  opacity: 1;
  visibility: visible;
}

.kb-image-lightbox img {
  max-width: min(92vw, 1280px);
  max-height: 88vh;
  border-radius: 18px;
  box-shadow: 0 28px 80px rgba(0, 0, 0, 0.42);
  cursor: zoom-in;
  transition: transform 0.12s ease-out;
  transform-origin: center center;
}

.kb-image-lightbox-close {
  position: fixed;
  top: 18px;
  right: 18px;
  width: 38px;
  height: 38px;
  border: 1px solid rgba(255, 255, 255, 0.24);
  border-radius: 999px;
  background: rgba(15, 23, 42, 0.58);
  color: #fff;
  font-size: 24px;
  line-height: 1;
  cursor: pointer;
  transition: background-color 0.18s ease, transform 0.18s ease;
}

.kb-image-lightbox-close:hover {
  background: rgba(37, 99, 235, 0.82);
  transform: translateY(-1px);
}
```

- [ ] **Step 2: Verify CSS syntax and site build**

Run:

```bash
hugo --quiet
```

Expected: command exits with code 0 and VS Code reports no CSS syntax errors in `assets/css/extended/custom.css`.

## Task 4: Manual browser verification

**Files:**
- No file changes.

- [ ] **Step 1: Start the local Hugo server**

Run:

```bash
hugo server --bind 127.0.0.1 --port 1313
```

Expected: server starts and reports a local URL at `http://127.0.0.1:1313/`.

- [ ] **Step 2: Verify code copy buttons**

Open an article page that contains a fenced code block.

Expected:
- A copy button appears on each code block.
- Clicking it copies the code text to the clipboard.
- The button label changes to the copied state briefly, using PaperMod's existing behavior.

- [ ] **Step 3: Verify image preview**

Open an article page that contains an image inside `.post-content`.

Expected:
- The cursor changes to zoom-in over the image.
- Clicking the image opens a dark full-screen preview.
- Scrolling up over the preview enlarges the image.
- Scrolling down shrinks the image, stopping at 0.5x.
- The image stops enlarging at 4x.
- Clicking the background closes the preview.
- Pressing Escape closes the preview.
- Clicking the close button closes the preview.

- [ ] **Step 4: Stop the server**

Press `Ctrl+C` in the terminal running Hugo server.

Expected: server exits cleanly.
