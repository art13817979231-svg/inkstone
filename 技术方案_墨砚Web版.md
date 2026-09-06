# 墨砚 Inkstone · Web 版技术方案 v1.0

> 对应业务 PRD v1.0（墨砚Inkstone_产品PRD_v1.0.md）。本文档为技术方案 + 模块拆分 + 接口清单，供后续原生版开发复用同一架构思想。

---

## 1. 总体架构

| 层 | 选型 | 理由 |
|---|---|---|
| 形态 | 纯前端单页应用（SPA，单文件 index.html） | 零后端、零服务器成本，与 PRD「iCloud 直连」哲学一致（浏览器侧用 IndexedDB 替代） |
| 持久化 | IndexedDB（books / anns 两个 store）+ localStorage（偏好） | 结构化、大文件 Blob 支持、跨会话 |
| 文档解析 | TXT/MD 自研；EPUB 自研（JSZip 解包 + DOMParser 重排版）；PDF 用 pdf.js（CDN 按需加载） | 对齐 PRD「纸感排版引擎：忽略原书硬编码样式」 |
| AI 能力 | 本地抽取式摘要（TF-IDF 句子打分）+ 段落检索问答为兜底；可插拔 OpenAI 兼容 API（用户自填 baseURL/key/model） | PRD「AI 只做辅助不替你读书」；无 Key 也完整可用 |
| 部署 | 静态托管（WorkBuddy Sites 发布为手机可访问链接） | 用户交付偏好：所有 HTML 必须有可访问链接 |

## 2. 模块拆分（单文件内的逻辑模块）

| 模块 | 职责 | 关键接口（函数） |
|---|---|---|
| M1 基础设施 | 工具函数、Toast、脚本懒加载 | `$(sel)` `uid()` `loadScript(src)` `toast(msg)` |
| M2 存储层 | IndexedDB 封装 | `dbPut(store,obj)` `dbGetAll(store)` `dbGet(store,id)` `dbDelete(store,id)` |
| M3 解析器 | 四格式 → 统一内容模型 | `parseTxt(file)` `parseMd(text)` `parseEpub(buf)` `parsePdf(buf)` |
| M4 书架 | 导入、封面生成、管理、统计 | `importFiles(files)` `renderLibrary()` `makeCover(title)` `deleteBook(id)` |
| M5 阅读器 | 净空界面、章节导航、进度、排版调节、4 主题 | `openBook(id)` `renderChapter(i)` `applyTypo()` `saveProgress()` |
| M6 批注 | 选区高亮（4色+下划线）、文字批注、PDF 页级批注、侧栏 | `createAnnotation(sel,color,note)` `applyMarks(root,anns)` `renderAnnotSidebar()` |
| M7 检索 | 全文 + 批注双轨、跨书检索 | `runSearch(query)` `jumpToResult(bookId,chapter,idx)` |
| M8 AI 面板 | 摘要 / 问答 / 关键词 | `localSummary(text)` `localQA(query)` `llmChat(messages)` |
| M9 导出 | Markdown（批注+原文合并）下载 | `exportMarkdown(bookId,mode)` |
| M10 统计 | 阅读时长累计、周报 | `tickReading()` `renderStats()` |

## 3. 统一内容模型（所有格式归一）

```js
BookRecord = {
  id, title, author, format: 'pdf'|'epub'|'txt'|'md',
  cover,            // dataURL（epub 用内嵌封面；其余 canvas 生成 Swiss 风封面）
  addedAt, lastOpened,
  progress: { chapterIndex, scrollPct, page },   // pdf 用 page
  readMs, readMsWeek, weekKey,
  blob,             // 原始文件（pdf 重渲染 / epub 复解析）
  content: {
    chapters: [ { title, blocks: [ {type:'h1'|'h2'|'p'|'quote'|'code'|'ul'|'ol'|'hr', text|items} ] } ],
    pageCount      // 仅 pdf
  }
}
Annotation = {
  id, bookId, chapterIndex,      // pdf 批注用 page 代替
  page,                          // pdf
  text,                          // 被标注原文（用于回显与检索定位）
  color: 'cinnabar'|'indigo'|'ochre'|'moss',
  type: 'highlight'|'underline'|'pagenote',
  note, createdAt
}
```

## 4. 关键技术决策

1. **EPUB 重排版**：解包后只取语义结构（标题/段落/引用/列表），丢弃原书全部内联样式 → 呼应 PRD「自研纸感排版引擎」。
2. **高亮回显**：章节渲染后基于 TreeWalker 拼接全文 → `indexOf` 定位 → `Range.surroundContents` 包裹 `<mark>`，天然支持跨内联元素容错。
3. **PDF V1 限制（已告知的风险）**：canvas 渲染阅读 + 导入期全量抽文本层（供检索/摘要/问答）；批注为**页级笔记**而非选区高亮（选区高亮留给原生版 PencilKit）。
4. **AI 双轨**：本地算法保证离线可用；设置面板填入 OpenAI 兼容端点后，摘要/问答自动升级为 LLM 合成（检索段落作为上下文注入，即轻量 RAG）。CORS 取决于所选端点，属用户侧配置风险。
5. **依赖**：仅 pdf.js 3.11 与 JSZip 3.10 两个 CDN 脚本，均为按需懒加载；TXT/MD/检索/导出离线零依赖。

## 5. 界面结构

```
顶栏：墨砚 logo · 搜索 · 设置（AI/偏好）
书架视图：统计条（本周时长/在读/批注数）+ 封面网格 + 导入卡
阅读视图：净空模式（默认无 UI，双击/滚顶唤出）
  ├ 左抽屉：目录
  ├ 右侧栏：批注列表
  ├ 右浮面板：AI（摘要/问答）
  └ 底栏：章节导航 · 排版调节（字号12级/行距5级/段距3级）· 主题4种 · 导出
键盘：⌘F 检索 · ←/→ 翻章 · Esc 关闭
```

## 6. 与业务 PRD 的差异说明（Web MVP 裁剪）

| PRD 条目 | Web MVP 处理 |
|---|---|
| iCloud/Handoff/Family Sharing/Spotlight/Siri | 不适用（原生能力），由 IndexedDB + 发布链接跨设备访问替代 |
| 语音批注、Apple Pencil 手写 | 不做，列为原生版专属 |
| Notion/Obsidian 直连导出 | V1 给 Markdown 文件导出（Obsidian 可直接导入），API 直连留 V2 |
| CMYK 色板 / 网格线彩蛋 | 高亮 4 色采用 Pantone 灵感色（朱砂/黛蓝/藤黄/苔绿）落地 |

## 7. 部署注意事项

- 静态单文件，任何静态托管可用；发布链接已按交付偏好生成。
- HTTPS 下 IndexedDB 与剪贴板均正常；CDN 依赖首次使用 PDF/EPUB 时需联网。
- localStorage 键：`inkstone.prefs`（排版/主题）、`inkstone.llm`（AI 端点配置，仅存本机浏览器）。
