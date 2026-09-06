# 墨砚 Inkstone · 接力文档

> 新对话开场请直接复制下面这段，再补上你要做的事。

## 开场白（复制这段）

```
工作空间：/Users/xia/Desktop/ppt/2026-09-06-06-14-11/

先读 /Users/xia/Desktop/ppt/2026-09-06-06-14-11/HANDOFF_墨砚.md，
再读 /Users/xia/Desktop/ppt/2026-09-06-06-14-11/.workbuddy/memory/ 下日期最新的那份日志，
然后继续墨砚阅读器的开发。本次要做的事：<在这里写你的需求>
```

---

## 一句话现状

纯前端单文件 AI 阅读器 MVP，按《墨砚Inkstone_产品PRD_v1.0.md》落地（PRD 原为 iOS/macOS 原生，本项目按工作区背景做 Web 版）。**自用路线优先**——文件夹/标签等面向陌生用户的功能暂缓。

## 关键路径

| 内容 | 路径 |
|---|---|
| 源码（单文件，2055 行） | `/Users/xia/Desktop/ppt/2026-09-06-06-14-11/inkstone/index.html` |
| PRD | `/Users/xia/Desktop/ppt/2026-09-06-06-14-11/墨砚Inkstone_产品PRD_v1.0.md` |
| 技术方案 | `/Users/xia/Desktop/ppt/2026-09-06-06-14-11/技术方案_墨砚Web版.md` |
| 工作日志 | `/Users/xia/Desktop/ppt/2026-09-06-06-14-11/.workbuddy/memory/2026-09-06.md` |
| 线上地址 | https://0d50484873dc465c8bbd791058aeb89a.app.workbuddy.link |

**工作空间必须选** `/Users/xia/Desktop/ppt/2026-09-06-06-14-11/`，否则上面的记忆和代码都找不到。

## 代码结构（M1–M11，按行号定位）

```
9    设计系统（Swiss 极简：纸白#fafaf7 / 墨黑#1a1a1a / 朱砂#c8102e）
488  M1  基础工具        toast / loadScript / debounce
519  M2  存储层          IndexedDB: dbPut/dbGetAll/dbDelete/dbClear；prefs 与 LLM 配置走 localStorage
545  M3  文档解析        parseEpub / parsePdf / parseTxt / parseMd
904  M4  书架            importFiles / renderLibrary / 封面生成
972  M5  阅读器          4主题 / 排版调节 / 净空模式(5秒隐藏)
1145 M6  批注            四色高亮+下划线+文字批注；PDF 为页级 pagenote
1211 M7  检索            全文+批注双轨，跨书
1274 M8  AI              本地抽取式摘要 + IDF 问答；可插 OpenAI 兼容接口
1389 M9  导出            Markdown / Obsidian ZIP（自写 zipStore）
1629 M10 阅读计时
1639 M11 数据保全        exportBackup / restoreBackup
1754 选区批注交互
1855 面板 / 工具栏交互
2039 启动
```

## 已定决策（别推翻）

1. **自用优先**：华东明确"先自己用"。优先级排序 = 自用流水线（读 → 批注 → 导出 Obsidian → 写内容）> 面向陌生用户的秩序（文件夹/标签/排序）。
2. **零后端**：全部数据在本机 IndexedDB，无服务端。PRD 的 iCloud 同步是原生版的事。
3. **PRD Q1–Q4 全取 A**：命名墨砚 Inkstone / 买断+订阅双轨 / MVP 不含 AI 服务端依赖 / 首发不做多端原生。
4. **PDF 批注为页级笔记**：canvas 渲染不支持选区高亮，是 V1 已知限制（已写进技术方案）。

## 环境注意事项（踩过坑）

- **Puppeteer 启动必须加** `--no-sandbox --disable-dev-shm-usage --disable-gpu`，否则沙箱内连接随机断。
- 无头测试点工具栏按钮前要先 `mouse.move` 唤出——净空模式 5 秒自动隐藏会让按钮不可点。
- 批注侧栏容器是 `#ann-list`（面板是 `#annot-panel`），不是 `#annot-list`。
- `addAnnotation` 在 PDF 视图下只接受 `type === 'pagenote'`，其余直接 return。
- `S.books[0]` 可能是内置 md 样张（无 blob），断言前按 `format` 定位目标书。
- macOS 自带 `unzip` 命令行显示中文文件名为 `???`（BSD 版不支持 UTF-8 flag），非 bug；双击解压或用 `ditto -xk` 正常。
- pdf.js / jszip 走 cdnjs CDN 懒加载，首次导入 PDF/EPUB 需联网。
- **PDF 旧数据**：章节结构会自动迁移（`migratePdfChapters` 在 openBook 时补齐 `pageStart`），但正文文本层不会重抽——升级前导入的 PDF 若正文是部首字（搜「留白」搜不到「留⽩」），仍需重新导入一次才生效。
- 造测试素材：`pypdf`（装在 `/Users/xia/.workbuddy/binaries/python/envs/default`）可给 PDF 加书签；`puppeteer.page.pdf()` 生成的 PDF **不带**书签。

## 待办候选（按当前优先级，2026-09-06 第三轮后重排）

| 优先级 | 事项 | 说明 |
|---|---|---|
| 中 | Obsidian 按主题聚合导出 | 当前按章节。**取决于华东跑完第一本书后的实际感受，别提前做** |
| 中 | 书架排序 / 筛选 | 现只有添加顺序 |
| 低 | PDF 导入进度条 | ⚠️ **已实测降级**：150 页 PDF 导入仅 2.2 秒、主线程最长冻结 38ms，根本不卡。此前"会阻塞主线程"是未测先判的错误结论。真要做也只是为了消除等待焦虑，非性能问题 |
| 低 | 读完状态 | 统计里缺"读完几本"（PRD P1） |
| 低 | 检索索引 | 现为每书每章 indexOf，书库大了会慢 |
| 低 | 跨 span 高亮合并 | 一段文字若被 PDF 切成多个文本块，会生成多个相邻 `mark`，视觉上连续但 DOM 上是多个节点。仅影响导出时的块引用粒度，不影响观感 |

### 已完成 · 第四轮（PDF 划词高亮）

- **textLayer**：`renderPdfPage` 渲染 canvas 后叠加 pdf.js `renderTextLayer`（3.11 有此 API）。CSS 是官方 `text_layer` 的精简内联版（单文件不能引外部样式表），文字保持 `transparent` 以免与 canvas 原字重影，高亮走背景色。
- **划词链路**：`captureSel` 支持文本层并计算 `nth`（该文本在本页第几次出现，避免页眉/重复句标错位置）；`mouseup` 去掉 PDF 排除；`addAnnotation` 解除「PDF 只能 pagenote」限制。
- **高亮重绘**：`applyPdfMarks()` 按文本节点分组做字符级精确包裹（span 是 absolute，内部 inline 拆分不破坏定位）。
- **顺带解锁**：PDF 朗读（此前 `toggleSpeak` 显式禁用，现取文本层当前页）、划词检索（气泡新增「检索」按钮）、PDF 文字可直接 Ctrl/Cmd+C 复制。

⚠️ **关键设计约束（改这块前必读）**：
1. **文本层渲染后必须立刻 `normalizeCJK`**。文本层里躺的是部首字（`留⽩`），不归一化的话——选区捕获到「留⽩」、批注存「留白」、重绘时两者永远对不上，高亮静默失效。
2. **每次重绘前必须还原干净快照**（`S.pdfLayerHTML`）。若只跳过 `mark` 内文本去构建字符序列，序列会残缺，导致后续批注定位**累积漂移**（实测：第 2 条批注起 mark 数就从 3 变 4）。
3. **PDF 分支的重绘必须显式调 `applyPdfMarks()`**——`renderChapter` 对 PDF 第一行就 return。已踩两处：`deleteAnn`、批注编辑保存（`note-save`）。新增任何改批注的入口都要检查这条。

### 已完成 · 第三轮（PDF 书签 / 目录）

- `parsePdf` 读 `doc.getOutline()`，按书签把页面切成真实章节（带 `pageStart`/`pageEnd`/`level`）；无书签回退每页一章。
- 目录可点击跳转（此前 PDF 目录是死的：`renderChapter` 对 PDF 直接 return，且 1959 行的点击处理有 `goPage(i+1)` 旧时代特判，绕过章节）。
- 目录高亮、底部页码标签（此前从未被赋值，一直是空白）、顶部章节栏均可用。
- **页级批注的 `chapterIndex` 改为按当前页反查章节**（原用 `S.chapter`，而它只在点目录时更新，翻页不变 → 全书批注全归第一章）。
- Obsidian 导出：有书签的 PDF 不再输出单文件，改走与 EPUB 相同的多文件章节路径（主笔记 + 每章一文件 + 块引用 + 页码区间）。
- 旧数据兼容：`migratePdfChapters()` 在 openBook 时为无 `pageStart` 的旧 PDF 补齐区间；`curChapterIndex()` 另有兜底。

## 一条重要提醒

**在华东跑完第一本真书之前，不要加新功能。** 当前最该发生的事是：拿一本真在看的书走完整链路（导入 → 批注 → 导出 → Obsidian），用真实体感判断批注密度和导出粒度合不合手。这是自用路线唯一重要的验证。
