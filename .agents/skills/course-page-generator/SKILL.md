---
name: course-page-generator
description: 將主題或草稿轉換成標準講義資料夾，產生 content.md、config.yaml、index.html 與 OG 圖。支援 topic mode 與 draft mode，最後必須 build、generate-og，並更新 lectures/catalog.yaml。
---

# Course Page Generator

Use this skill when the user wants to generate, reorganize, or update a lecture page.

The project now has two input modes:

- `topic mode`: the user provides a topic only. Create a full lecture from the topic.
- `draft mode`: the user provides draft notes, a transcript, or rough content. Convert it into a polished lecture.

Both modes must output the same standard lecture structure.

## Output Layout

Topic-generated lectures go here:

```text
lectures/topic/<slug>/
```

Draft-generated lectures go here:

```text
lectures/draft/<slug>/
```

Every lecture directory must contain:

```text
config.yaml
content.md
index.html
assets/
  og-image.jpg
```

The project-level catalog must be updated:

```text
lectures/catalog.yaml
```

## Workflow

### Step 1: Decide Mode

Use `topic mode` when:

- The user gives only a title or topic.
- The user asks to design a new lecture.
- There is no long draft to preserve.

Use `draft mode` when:

- The user provides notes, a transcript, a README, or rough material.
- The main task is restructuring and polishing existing material.
- The output should preserve the user's source ideas.

If the mode is obvious, do not ask. Make the reasonable choice and proceed.

### Step 2: Choose Slug And Path

Use kebab-case for the slug.

Examples:

```text
OpenAI Codex 及 Claude Code -> openai-codex-claude-code
生成式 AI 資安 -> generative-ai-security
```

Path rules:

```text
topic mode -> lectures/topic/<slug>
draft mode -> lectures/draft/<slug>
```

### Step 3: Create Standard Files

Create or update:

- `content.md`
- `config.yaml`
- `assets/`

For draft mode, optionally keep the source as:

- `draft.md`
- `source.md`

Only keep a source file when it is useful for future editing.

### Step 4: Write `content.md`

Use this structure:

```markdown
# Main Title
> Lead text

[summary]
- Label **Key point** | Description
[/summary]

## Section

### Subsection
- Teaching point
- Teaching point

> **Insight Title**
> Insight content

[flow]
1. Step one
2. Step two
[/flow]
```

Guidelines:

- Keep the lecture teachable, not just informative.
- Use `##` for major teaching sections.
- Use `###` for teachable cards or focused subsections.
- Use `[summary]` near the beginning.
- Use `[flow]` for process explanations.
- Use insight boxes for core takeaways.
- Use prompt blocks for agent or CLI examples.
- Avoid leaving loose, unstructured paragraphs when a list, flow, or insight box would teach better.

### Step 5: Write `config.yaml`

At minimum:

```yaml
page:
  lang: zh-TW
  title: "Lecture title"
  badge: "Lecture | Category"
  hero_title: "Hero title"
  subtitle: "Short teaching subtitle"

seo:
  site_name: "Lecture title"
  type: "article"
  title: "SEO title"
  description: "SEO description"
  image: "https://<owner>.github.io/<repo>/<lecture-path>/assets/og-image.jpg"
  url: "https://<owner>.github.io/<repo>/<lecture-path>/"

quotes:
  opening:
    text: "Opening quote"
  closing:
    text: >
      Closing quote
```

Resolve GitHub Pages URLs from:

```bash
git remote get-url origin
```

For `https://github.com/yuben75/agent-skill-lecture-builder.git`, use:

```text
https://yuben75.github.io/agent-skill-lecture-builder/<lecture-path>/
```

### Step 6: Update Catalog

Update `lectures/catalog.yaml`:

```yaml
lectures:
  - slug: example
    type: draft
    title: "Lecture title"
    path: "lectures/draft/example"
    status: published
    description: "Short description."
```

Use `type: topic` or `type: draft`.

Recommended statuses:

- `published`
- `draft`
- `example`

### Step 7: Build

Run from the repo root:

```bash
node .agents/skills/course-page-generator/scripts/build.mjs <lecture-path>
```

Example:

```bash
node .agents/skills/course-page-generator/scripts/build.mjs lectures/topic/openai-codex-claude-code
```

### Step 8: Generate OG Image

Run:

```bash
node .agents/skills/course-page-generator/scripts/generate-og.mjs <lecture-path>
```

Example:

```bash
node .agents/skills/course-page-generator/scripts/generate-og.mjs lectures/topic/openai-codex-claude-code
```

If Puppeteer cannot find Chrome on Windows, use:

```powershell
$env:PUPPETEER_EXECUTABLE_PATH="C:\Program Files\Google\Chrome\Application\chrome.exe"
```

If Puppeteer tries to download Chrome during install and the machine already has Chrome:

```bash
PUPPETEER_SKIP_DOWNLOAD=true npm install
```

### Step 9: Verify

Verify:

- The lecture directory has `content.md`, `config.yaml`, `index.html`, and `assets/og-image.jpg`.
- `lectures/catalog.yaml` contains the lecture.
- `index.html` has the expected SEO URL and image URL.
- The page can be served through `dev.mjs` or opened as generated HTML.

## Existing Lectures

Current topic lectures:

- `lectures/topic/generative-ai-security`
- `lectures/topic/openai-codex-claude-code`

Current draft lecture:

- `lectures/draft/example`

## Reference Files

Use these files for syntax and rendering details:

- `reference/components.md`
- `reference/content-example.md`
- `reference/config-example.yaml`
- `reference/base.html`
