# AntFluid 与 RikkaHub 实时 Markdown 渲染对比

## 结论

如果目标是 **LLM 流式输出时，Markdown 表格、代码、公式边生成边稳定显示且界面不闪烁**，当前项目 **AntFluid** 的方案更适合。

如果目标是 **纯 Compose、易维护、UI 组件化、和应用主题/交互深度融合**，**RikkaHub** 的方案更现代、更好扩展，但在高频流式更新和复杂表格稳定性上不如 AntFluid 专项优化充分。

简要判断：

> **现阶段要解决“实时渲染表格且界面不闪烁”，AntFluid 更好。**  
> **要做长期 Compose 架构和可维护 UI，RikkaHub 更好。**  
> **最佳长期方向是把 AntFluid 的流式/缓存机制迁移成 Compose 分块渲染模型。**

---

## 1. AntFluid 的实现原理

AntFluid 采用的是：

```text
TextView + Markwon/CommonMark + Spannable + ReplacementSpan + 流式打印状态机
```

核心文件：

- `fluid-markdown/src/main/java/com/fluid/afm/markdown/widget/PrinterMarkDownTextView.java`
- `fluid-markdown/src/main/java/com/fluid/afm/markdown/MarkdownParser.java`
- `fluid-markdown/src/main/java/com/fluid/afm/markdown/MarkdownParserFactory.java`
- `markwon-ext-tables/src/main/java/io/noties/markwon/ext/tables/TablePlugin.java`
- `markwon-ext-tables/src/main/java/io/noties/markwon/ext/tables/TableRowSpan.java`
- `markwon-ext-tables/src/main/java/io/noties/markwon/ext/tables/TableRowsScheduler.java`

### 1.1 流式打印流程

`PrinterMarkDownTextView` 是核心入口。

普通设置 Markdown：

```java
mMarkdownParser.getMarkwon().setMarkdown(this, markdown);
```

对应位置：`fluid-markdown/src/main/java/com/fluid/afm/markdown/widget/PrinterMarkDownTextView.java:125-133`

流式打印开始时：

```java
mMarkdownParser.setPrintingState(true);
mParsedContentText = new SpannableStringBuilder(mMarkdownParser.getMarkwon().toMarkdown(mOriginText));
printing(startIndex, mChunkSize);
```

对应位置：`fluid-markdown/src/main/java/com/fluid/afm/markdown/widget/PrinterMarkDownTextView.java:143-176`

新增 token 到来时：

```java
mOriginText += content;
mParsedContentText = new SpannableStringBuilder(mMarkdownParser.getMarkwon().toMarkdown(mOriginText));
printing(mCurrentPrintIndex, mChunkSize);
```

对应位置：`fluid-markdown/src/main/java/com/fluid/afm/markdown/widget/PrinterMarkDownTextView.java:178-202`

打印过程由 `Handler` 节奏控制：

```java
MAIN_HANDLER.postDelayed(mPrintTask, mInterval);
```

对应位置：`fluid-markdown/src/main/java/com/fluid/afm/markdown/widget/PrinterMarkDownTextView.java:328-350`

它的关键点是：

1. 每次可以重新解析完整 Markdown；
2. 但实际显示只截取到当前打印 index；
3. 已显示部分通过 `Spannable` 和 span 复用保持稳定；
4. UI 是逐步露出，而不是整块内容频繁替换。

流程图：

```text
LLM token 到来
   |
appendPrinting()
   |
更新 mOriginText
   |
Markwon.toMarkdown() 生成完整 Spannable
   |
handleSpan() 截取当前可见部分并复制 span
   |
setParsedMarkdown(TextView, partialSpannable)
   |
TextView 继续稳定绘制
```

### 1.2 防闪烁关键：Span 截取与复用

`handleSpan()` 会从完整解析结果中复制已经显示范围内的 `CharacterStyle`：

```java
CharacterStyle[] spans = source.getSpans(0, end, CharacterStyle.class);
...
newSpannable.setSpan(span, spanStart, Math.min(spanEnd, end), flags);
```

对应位置：`fluid-markdown/src/main/java/com/fluid/afm/markdown/widget/PrinterMarkDownTextView.java:398-420`

最后设置文本时不是直接 `TextView.setText()`，而是走 Markwon 的 `setParsedMarkdown()`，这样插件的 `beforeSetText/afterSetText` 生命周期仍然生效：

```java
mMarkdownParser.getMarkwon().setParsedMarkdown(this, spanned);
```

对应位置：`fluid-markdown/src/main/java/com/fluid/afm/markdown/widget/PrinterMarkDownTextView.java:468-470`

Markwon 内部会：

```java
plugin.beforeSetText(textView, markdown);
textView.setText(markdown, bufferType);
plugin.afterSetText(textView);
```

对应位置：`markwon-core/src/main/java/io/noties/markwon/MarkwonImpl.java:117-141`

### 1.3 表格专项优化

AntFluid 对 Markwon 表格插件做了流式优化。

`TablePlugin` 实现了 `StreamOutStateObserver`：

```java
public class TablePlugin extends AbstractMarkwonPlugin implements StreamOutStateObserver
```

对应位置：`markwon-ext-tables/src/main/java/io/noties/markwon/ext/tables/TablePlugin.java:60`

当流式状态变化时：

```java
visitor.setStreamingOutput(isStreamingOutput);
if (!isStreamingOutput) {
    visitor.mTableSpanCache.clear();
}
```

对应位置：`markwon-ext-tables/src/main/java/io/noties/markwon/ext/tables/TablePlugin.java:84-89`

表格行 span 有缓存：

```java
public ConcurrentHashMap<Integer, ConcurrentHashMap<String, TableRowSpan>> mTableSpanCache
```

对应位置：`markwon-ext-tables/src/main/java/io/noties/markwon/ext/tables/TablePlugin.java:169`

渲染表格行时，会尝试复用缓存的 `TableRowSpan`：

```java
TableRowSpan cachedSpan = getCachedSpan(tableIndex, length);
...
if (useCachedSpan && cachedSpan != null && !isHideTitle) {
    final TableRowSpan span = getCachedSpan(tableIndex, length);
    visitor.setSpans(start, span);
}
```

对应位置：`markwon-ext-tables/src/main/java/io/noties/markwon/ext/tables/TablePlugin.java:287-337`

如果需要创建新表格行 span，则在流式状态下缓存：

```java
final TableRowSpan span = new TableRowSpan(...);
...
if (cacheSpan) {
    cacheSpan(tableIndex, length, span);
}
```

对应位置：`markwon-ext-tables/src/main/java/io/noties/markwon/ext/tables/TablePlugin.java:347-375`

这使得表格在边生成边显示时，已稳定的行不会频繁销毁和重建。

### 1.4 表格绘制方式

AntFluid 的表格不是普通 ViewGroup，也不是 Compose 表格，而是 `ReplacementSpan` 自绘。

`TableRowSpan`：

```java
public class TableRowSpan extends ReplacementSpan
```

对应位置：`markwon-ext-tables/src/main/java/io/noties/markwon/ext/tables/TableRowSpan.java:39`

它在 `draw()` 里绘制：

- 背景；
- 边框；
- 圆角；
- 单元格分隔线；
- 单元格文本。

对应位置：`markwon-ext-tables/src/main/java/io/noties/markwon/ext/tables/TableRowSpan.java:227-429`

单元格文本用 `StaticLayout` 预排版：

```java
final Layout layout = new StaticLayout(...);
```

对应位置：`markwon-ext-tables/src/main/java/io/noties/markwon/ext/tables/TableRowSpan.java:525-560`

### 1.5 表格刷新合并

表格行绘制过程中，如果测量结果变化，会触发 invalidation。但 AntFluid 没有每次立即刷新，而是合并刷新：

```java
view.removeCallbacks(runnable);
view.post(runnable);
```

对应位置：`markwon-ext-tables/src/main/java/io/noties/markwon/ext/tables/TableRowsScheduler.java:36-53`

这能减少表格布局过程中的抖动。

---

## 2. RikkaHub 的实现原理

RikkaHub 采用的是：

```text
Compose + IntelliJ Markdown AST + 后台解析 + Compose 组件树渲染
```

核心文件：

- `app/src/main/java/me/rerere/rikkahub/ui/components/richtext/Markdown.kt`
- `app/src/main/java/me/rerere/rikkahub/ui/components/richtext/MarkdownNew.kt`
- `app/src/main/java/me/rerere/rikkahub/ui/components/table/DataTable.kt`
- `app/src/main/java/me/rerere/rikkahub/ui/components/message/ChatMessage.kt`

### 2.1 MarkdownBlock 主流程

RikkaHub 使用 JetBrains Markdown parser：

```kotlin
private val flavour by lazy {
    GFMFlavourDescriptor(makeHttpsAutoLinks = true, useSafeLinks = true)
}

private val parser by lazy {
    MarkdownParser(flavour)
}
```

对应位置：`app/src/main/java/me/rerere/rikkahub/ui/components/richtext/Markdown.kt:110-118`

解析函数：

```kotlin
private fun parseMarkdown(content: String): MarkdownParseResult {
    val preprocessed = preProcess(content)
    val astTree = parser.buildMarkdownTreeFromString(preprocessed)
    return MarkdownParseResult(preprocessed, astTree, astTree.containsHtml())
}
```

对应位置：`app/src/main/java/me/rerere/rikkahub/ui/components/richtext/Markdown.kt:224-228`

`MarkdownBlock()` 内部会先持有当前解析结果：

```kotlin
var (data, setData) = remember { mutableStateOf(parseMarkdown(content)) }
```

然后监听 content 变化，后台重新解析：

```kotlin
snapshotFlow { updatedContent }
    .distinctUntilChanged()
    .mapLatest { parseMarkdown(it) }
    .flowOn(Dispatchers.Default)
    .collect { setData(it) }
```

对应位置：`app/src/main/java/me/rerere/rikkahub/ui/components/richtext/Markdown.kt:231-249`

流程图：

```text
content state 更新
   |
snapshotFlow 监听
   |
distinctUntilChanged 去重
   |
Dispatchers.Default 后台 parse AST
   |
解析完成后 setData
   |
Compose 重组 MarkdownNode 树
```

### 2.2 HTML 路径

如果 AST 中包含 HTML，RikkaHub 会切到 `MarkdownNew()`：

```kotlin
if (data.hasHtml) {
    MarkdownNew(...)
} else {
    MarkdownNode(...)
}
```

对应位置：`app/src/main/java/me/rerere/rikkahub/ui/components/richtext/Markdown.kt:251-270`

`MarkdownNew()` 会先生成 HTML：

```kotlin
return HtmlGenerator(preprocessed, tree, flavour).generateHtml()
```

对应位置：`app/src/main/java/me/rerere/rikkahub/ui/components/richtext/MarkdownNew.kt:115-119`

然后后台更新 HTML：

```kotlin
snapshotFlow { updatedContent }
    .distinctUntilChanged()
    .mapLatest { generateMarkdownHtml(it) }
    .flowOn(Dispatchers.Default)
    .collect { html = it }
```

对应位置：`app/src/main/java/me/rerere/rikkahub/ui/components/richtext/MarkdownNew.kt:130-144`

最后用 Jsoup 转成节点树并渲染成 Compose：

```kotlin
val document = remember(html) {
    runCatching { Jsoup.parse(html) }.getOrElse { Jsoup.parse("") }
}
```

对应位置：`app/src/main/java/me/rerere/rikkahub/ui/components/richtext/MarkdownNew.kt:146-148`

### 2.3 表格渲染

在 AST 路径中，遇到 GFM 表格：

```kotlin
GFMElementTypes.TABLE -> {
    TableNode(node = node, content = content, modifier = modifier)
}
```

对应位置：`app/src/main/java/me/rerere/rikkahub/ui/components/richtext/Markdown.kt:503-505`

`TableNode()` 会提取表头、行和单元格：

```kotlin
val headerNode = node.children.find { it.type == GFMElementTypes.HEADER }
val rowNodes = node.children.filter { it.type == GFMElementTypes.ROW }
val columnCount = headerNode?.children?.count { it.type == GFMTokenTypes.CELL } ?: 0
```

对应位置：`app/src/main/java/me/rerere/rikkahub/ui/components/richtext/Markdown.kt:825-844`

每个单元格再用 `MarkdownBlock()` 渲染：

```kotlin
MarkdownBlock(
    content = if (columnIndex < rowData.size) rowData[columnIndex] else "",
)
```

对应位置：`app/src/main/java/me/rerere/rikkahub/ui/components/richtext/Markdown.kt:847-864`

最后交给 `DataTable()`：

```kotlin
DataTable(
    headers = headers,
    rows = rowComposables,
    columnMinWidths = List(columnCount) { 80.dp },
    columnMaxWidths = List(columnCount) { 200.dp },
    outerBorder = null,
)
```

对应位置：`app/src/main/java/me/rerere/rikkahub/ui/components/richtext/Markdown.kt:953-959`

在 HTML 路径中，`MarkdownNew()` 也会把 HTML `<table>` 转成 `DataTable()`：

```kotlin
private fun HtmlTable(element: Element, onClickCitation: (String) -> Unit) {
    ...
    DataTable(...)
}
```

对应位置：`app/src/main/java/me/rerere/rikkahub/ui/components/richtext/MarkdownNew.kt:589-632`

### 2.4 DataTable 自定义布局

`DataTable()` 是 Compose 自定义布局：

```kotlin
SubcomposeLayout { constraints -> ... }
```

对应位置：`app/src/main/java/me/rerere/rikkahub/ui/components/table/DataTable.kt:64`

它分两阶段测量：

1. 第一阶段：自然尺寸测量，估算列宽和行高；
2. 第二阶段：固定列宽和统一行高重新测量；
3. 最后手动 place 每个 cell。

关键位置：

- 第一阶段测量：`app/src/main/java/me/rerere/rikkahub/ui/components/table/DataTable.kt:78-119`
- 行高计算：`app/src/main/java/me/rerere/rikkahub/ui/components/table/DataTable.kt:121-128`
- 第二阶段测量：`app/src/main/java/me/rerere/rikkahub/ui/components/table/DataTable.kt:130-167`
- 放置 cell：`app/src/main/java/me/rerere/rikkahub/ui/components/table/DataTable.kt:174-190`

---

## 3. 两者核心区别

| 维度 | AntFluid | RikkaHub |
|---|---|---|
| UI 技术 | Android `TextView` | Jetpack Compose |
| Markdown 渲染 | Markwon/CommonMark + Spannable | IntelliJ Markdown AST + Compose |
| 表格实现 | `ReplacementSpan` 自绘表格行 | `SubcomposeLayout` 表格组件 |
| 流式输出模型 | 有显式 `startPrinting/appendPrinting` 状态机 | content state 变化后整体重新 parse |
| 表格流式优化 | 有 `TableRowSpan` 缓存、streaming state、刷新合并 | 无专门行级缓存，主要依赖 Compose 重组 |
| 防闪烁重点 | 已显示内容连续、旧 span 复用、表格行稳定 | 后台解析、旧 UI 保持到新 state ready |
| 性能倾向 | 长文本和高频流式更稳定 | 普通 Markdown 和组件交互更自然 |
| 维护性 | Span/Canvas/Markwon 插件复杂 | Compose 代码更直观 |
| 扩展交互 | Span 内复杂交互较麻烦 | 原生 Composable，交互扩展容易 |
| 适合场景 | LLM 流式输出、聊天打字机、表格边生成边稳定显示 | Compose App、静态或低频更新 Markdown、富交互 UI |

---

## 4. 为什么 AntFluid 更不容易在流式表格中闪烁

流式表格最大的问题是 Markdown 表格经常处于“不完整状态”：

```markdown
| A | B |
|---|---|
| 1 | 2 |
| 3 | ...
```

在 token 还没生成完时，parser 可能一会儿识别为表格，一会儿识别为普通文本，或者表格列数、行高、宽度反复变化。

AntFluid 的处理方式是：

```text
已显示内容尽量不变
新增内容按打印节奏露出
已稳定表格行复用 TableRowSpan
当前行完成后再更新缓存
多次表格刷新合并
```

尤其是这几个机制对不闪烁很关键：

1. `setPrintingState(true)` 通知插件进入流式模式；
2. `TablePlugin` 在流式模式下缓存 `TableRowSpan`；
3. 旧行优先复用缓存 span；
4. 不完整表格行通过 `findCurrentTableEnd()` 判断是否适合创建新 span；
5. `TableRowsScheduler` 合并 invalidation。

所以 AntFluid 解决的是：

> 流式过程中视觉连续、旧内容稳定、表格行级别减少抖动。

---

## 5. 为什么 RikkaHub 更易维护但流式表格专项能力弱一些

RikkaHub 的优势在于架构现代：

```text
Markdown AST
   |
Composable Node
   |
DataTable / Text / CodeBlock / Image / Math
```

它的优点是：

1. 完全 Compose 原生；
2. 主题、动画、点击、选择、布局都好接；
3. 表格是普通 Composable，复制、下载、横向滚动容易加；
4. 后台 parse 避免 UI 线程卡顿；
5. 不会像 WebView 那样整页 reload 白屏。

但它对高频流式表格有天然压力：

```text
content 高频变化
   |
完整 Markdown 重新 parse
   |
完整 AST state 替换
   |
MarkdownNode 树重组
   |
表格重新提取行列
   |
DataTable 两阶段重新测量
   |
单元格 MarkdownBlock 可能继续递归 parse
```

所以它通常不会“白屏闪”，但复杂表格高频更新时可能出现：

- 行高变化；
- 列宽变化；
- 表格识别状态变化；
- 组件重组导致的轻微跳动；
- SubcomposeLayout 测量开销较大。

---

## 6. 推荐方向

### 6.1 如果当前目标是解决实时 Markdown 表格不闪

优先选择 AntFluid 现有路线：

```text
TextView + Markwon + Spannable + TableRowSpan cache
```

这条路线已经有表格流式专项优化，不建议直接用 RikkaHub 的 `MarkdownBlock + DataTable` 替换。

### 6.2 如果目标是长期迁移到 Compose

建议不要简单照搬 RikkaHub，而是做混合设计：

```text
AntFluid 的流式状态机 / 表格缓存思想
        +
RikkaHub 的 Compose 组件化渲染
        |
        v
Compose 分块流式 Markdown Renderer
```

推荐模型：

```text
LLM token stream
   |
Markdown block segmenter
   |
按 block 判断稳定性
   |
已完成 block：缓存 AST/Composable model，不再频繁重组
   |
当前生成 block：允许高频更新
   |
表格 block：按 row 级别缓存
   |
DataTable：缓存列宽/行高，避免全表反复测量
```

### 6.3 Compose 版优化建议

如果要把 RikkaHub 方案改造成更适合流式的版本，可以考虑：

1. **分块解析**：按段落、代码块、表格块拆分，不要每 token 全文 parse。
2. **稳定块缓存**：已闭合的 Markdown block 不再重新 parse。
3. **表格行缓存**：表头和已完成行稳定后固定，当前行单独更新。
4. **未闭合表格保护**：当前表格未闭合时，优先保持旧表格结构，避免表格/纯文本之间反复切换。
5. **列宽缓存**：DataTable 的列宽不要每次从零计算。
6. **更新节流**：token 高频到来时，UI 层按 16ms/32ms/50ms 合并更新。
7. **稳定 key**：给 message block、table row、cell 提供稳定 key，降低 Compose 重组范围。
8. **避免 cell 递归完整 MarkdownBlock**：表格单元格可以使用轻量 inline renderer，避免每个 cell 都单独走完整 parse。

---

## 7. 最终评价

AntFluid 更像：

```text
流式 Markdown 渲染引擎
```

RikkaHub 更像：

```text
Compose Markdown UI 组件库
```

因此：

- 对 **LLM 聊天流式输出**：AntFluid 更强。
- 对 **Compose 应用 UI 一致性和维护性**：RikkaHub 更好。
- 对 **长期最优方案**：建议把 AntFluid 的流式缓存机制迁移成 Compose 分块渲染架构，而不是二选一。