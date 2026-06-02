# Agent Skill Lecture Builder

這個專案用固定的 Markdown 與 config 格式生成單頁講義網站，最後輸出 `index.html` 與 `assets/og-image.jpg`。

目前支援兩種講義來源：

- `topic`：只提供主題，由 AI agent 設計完整講義，例如 `generative-ai-security`、`openai-codex-claude-code`
- `draft`：提供草稿或非結構化筆記，由 AI agent 整理成正式講義，例如 `example`

## Directory

```text
agent-skill-lecture-builder/
  .agents/
    skills/
      course-page-generator/
        SKILL.md
        scripts/
          build.mjs
          dev.mjs
          generate-og.mjs
        reference/
          base.html
          components.md
          config-example.yaml
          content-example.md
  config/
    global.yaml
    assets/
      author.jpeg
  lectures/
    catalog.yaml
    topic/
      generative-ai-security/
      openai-codex-claude-code/
    draft/
      example/
  package.json
```

每份講義都維持同一種輸出格式：

```text
<lecture-dir>/
  config.yaml
  content.md
  index.html
  assets/
    og-image.jpg
```

## Lecture Catalog

`lectures/catalog.yaml` 是講義索引，記錄每份講義的分類、標題、slug、路徑與狀態。

新增講義時請同步更新 catalog：

```yaml
lectures:
  - slug: openai-codex-claude-code
    type: topic
    title: "OpenAI Codex 及 Claude Code"
    path: "lectures/topic/openai-codex-claude-code"
    status: published
```

## Quick Start

安裝依賴：

```bash
npm install
```

如果本機已經有 Chrome，且不想讓 Puppeteer 下載瀏覽器，可以使用：

```bash
PUPPETEER_SKIP_DOWNLOAD=true npm install
```

Windows PowerShell 若擋住 `npm.ps1`，可改用：

```powershell
npm.cmd install
```

## Build

建立單份講義：

```bash
node .agents/skills/course-page-generator/scripts/build.mjs lectures/topic/openai-codex-claude-code
```

產生 OG 圖：

```bash
node .agents/skills/course-page-generator/scripts/generate-og.mjs lectures/topic/openai-codex-claude-code
```

啟動本機預覽：

```bash
node .agents/skills/course-page-generator/scripts/dev.mjs lectures/topic/openai-codex-claude-code --port 8080
```

## npm Scripts

```bash
npm run build:example
npm run build:generative-ai-security
npm run build:openai-codex-claude-code

npm run dev:example
npm run dev:generative-ai-security
npm run dev:openai-codex-claude-code

npm run og:example
npm run og:generative-ai-security
npm run og:openai-codex-claude-code
```

## Two Generation Modes

### Topic Mode

使用者只提供主題，AI agent 需要自行設計講義結構、內容節奏、範例、提示詞與實作練習。

輸出位置：

```text
lectures/topic/<slug>/
```

範例：

```text
lectures/topic/generative-ai-security/
lectures/topic/openai-codex-claude-code/
```

### Draft Mode

使用者提供草稿、逐字稿、筆記或非結構化內容，AI agent 主要負責整理、補齊段落、轉成標準 Markdown 格式，並產生正式講義頁。

輸出位置：

```text
lectures/draft/<slug>/
```

範例：

```text
lectures/draft/example/
```

## Markdown Format

`content.md` 使用 course page generator 支援的 Markdown 元件：

~~~markdown
# 主標題
> 開場導言

[summary]
- 標籤 **重點** | 說明
[/summary]

## 章節

### 小節
- 條列內容
- 條列內容

> **Insight 標題**
> Insight 內容

[flow]
1. 第一步
2. 第二步
[/flow]

```prompt [label="Prompt 範例"]
請先閱讀 repo，再提出修改計畫。
```
~~~

更多元件請參考：

- `.agents/skills/course-page-generator/reference/components.md`
- `.agents/skills/course-page-generator/reference/content-example.md`

## Config

每份講義的 `config.yaml` 會覆蓋 `config/global.yaml`。

最少需要：

```yaml
page:
  lang: zh-TW
  title: "講義標題"
  badge: "分類標籤"
  hero_title: "Hero 標題"
  subtitle: "講義副標題"

seo:
  title: "SEO 標題"
  description: "SEO 描述"
  image: "https://example.com/path/assets/og-image.jpg"
  url: "https://example.com/path/"

quotes:
  opening:
    text: "開場引用"
  closing:
    text: >
      結尾引用
```

## Recommended Workflow

1. 決定模式：`topic` 或 `draft`
2. 決定 slug：使用 kebab-case
3. 建立講義資料夾
4. 產生 `content.md`
5. 產生 `config.yaml`
6. 更新 `lectures/catalog.yaml`
7. 執行 build
8. 執行 OG 圖生成
9. 檢查 `index.html`

## Notes

- `node_modules/` 不應提交
- `index.html` 是 build 產物，但目前會提交，方便 GitHub Pages 直接提供靜態頁
- `assets/og-image.jpg` 是生成產物，也會提交，方便社群分享預覽
- 如果 `generate-og.mjs` 找不到 Chrome，可設定 `PUPPETEER_EXECUTABLE_PATH`

Windows 範例：

```powershell
$env:PUPPETEER_EXECUTABLE_PATH="C:\Program Files\Google\Chrome\Application\chrome.exe"
node .agents/skills/course-page-generator/scripts/generate-og.mjs lectures/topic/openai-codex-claude-code
```
