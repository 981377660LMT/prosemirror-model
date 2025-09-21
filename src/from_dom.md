好的，我们来挑战 prosemirror-model 中最庞大、最复杂的文件之一：`from_dom.ts`。如果说 `to_dom.ts` 是将 ProseMirror 模型“翻译”成 DOM，那么 from_dom.ts 就是反向操作：它将一个混乱、不规范的 DOM 树，“解析”成一个结构严谨、符合 Schema 规则的 ProseMirror `Node` 或 `Slice`。

这个过程被称为**反序列化 (Deserialization)** 或**解析 (Parsing)**。这是实现从剪贴板粘贴、拖放 HTML 内容或从服务器加载 HTML 内容等功能的基石。

### 宏观理解：`DOMParser` 的巨大挑战

解析 HTML 是一项极其艰巨的任务，原因如下：

1.  **HTML 是“宽容”的**：浏览器会尽力渲染不规范的 HTML，比如 `<p><div>abc</div></p>`。但 ProseMirror 的模型是**严格**的，它不允许一个 `paragraph` 包含 `div`。
2.  **结构是隐式的**：HTML 中，`<strong>abc</strong>` 是一个 `<strong>` 元素。但在 ProseMirror 中，“粗体”是一个附加在 `text` 节点上的 `Mark`，而不是一个结构节点。
3.  **上下文很重要**：一个 `<li>` 标签只有在 `<ul>` 或 `<ol>` 内部时才有意义。解析器需要维护上下文信息。
4.  **规则是多样的**：一个 `<b>` 标签可能对应 `strong` 标记，一个 `style="font-weight: bold"` 的 `<span>` 也可能对应 `strong` 标记。解析器需要一个灵活的规则系统。

`DOMParser` 就是为了解决这些挑战而设计的。它的核心是一个**基于规则的、状态机驱动的解析引擎**。

---

### 第一部分：`ParseRule` - 解析的规则蓝图

与 `DOMOutputSpec` 类似，解析过程也是由一套规则驱动的。这些规则定义在 `Schema` 的 `NodeSpec` 和 `MarkSpec` 的 `parseDOM` 属性中。`ParseRule` 有两种主要类型：

#### 1. `TagParseRule` (匹配 DOM 标签)

- `tag`: 一个 CSS 选择器，如 `"p"`, `"img[src]"`, `"div.my-class"`。这是匹配的入口。
- `node` / `mark` / `ignore`: 匹配成功后做什么？
  - `node`: 创建一个指定名称的 ProseMirror `Node`。
  - `mark`: 创建一个指定名称的 `Mark`，并将其应用于后续内容。
  - `ignore`: 完全忽略这个 DOM 节点及其内容。
- `getAttrs(domNode)`: 一个函数，可以从 DOM 节点上提取属性。例如，从 `<img>` 的 `src` 属性中获取值。如果返回 `false`，则表示此规则不匹配。
- `contentElement`: 对于非叶子节点，其子节点内容可能不在 DOM 元素的直接子代中。此属性可以是一个 CSS 选择器，用于找到真正的内容容器。例如，对于一个自定义的 `figure` 组件，内容可能在 `figcaption` 中。
- `context`: 一个强大的功能，用于限制此规则只在特定的父节点上下文中生效。例如，`"blockquote/paragraph/"` 表示只有当解析器正在解析 `blockquote` 内部的 `paragraph` 时，此规则才有效。
- `priority`: 规则的优先级。数字越大，越先被尝试。

#### 2. `StyleParseRule` (匹配内联样式)

- `style`: 一个 CSS 属性名，如 `"font-weight"`，或者一个 "属性=值" 对，如 `"font-weight=bold"`。
- `mark`: 匹配成功后，应用哪个 `Mark`。
- `getAttrs(styleValue)`: 从样式值中提取属性。

---

### 第二部分：`DOMParser` - 规则的管理者

`DOMParser` 类本身相对简单。它的主要职责是管理规则。

- `constructor(schema, rules)`: 接收 `schema` 和一个 `ParseRule` 数组。它会将规则按 `tag` 和 `style` 分类，并按优先级排序，以备后续快速查找。
- `static fromSchema(schema)`: 最常用的工厂方法。它会自动从 `schema` 中收集所有 `parseDOM` 规则，并创建一个缓存的 `DOMParser` 实例。
- `parse(dom, options)`: 解析一个 DOM 节点，返回一个完整的 ProseMirror `Node`。
- `parseSlice(dom, options)`: 解析一个 DOM 节点，返回一个 `Slice`。这对于处理剪贴板内容非常有用，因为它保留了内容的“开放”边界。
- `matchTag` / `matchStyle`: 内部方法，用于根据给定的 DOM 节点或样式，在规则列表中查找第一个匹配的规则。

---

### 第三部分：`ParseContext` - 解析状态机

这是整个文件的核心和灵魂。`ParseContext` 是一个复杂的状态机，它在遍历 DOM 树的过程中，维护着正在构建的 ProseMirror 节点树的状态。

#### 1. 核心数据结构：`this.nodes` 栈

`ParseContext` 的核心是一个名为 `nodes` 的栈。栈中的每一个元素都是一个 `NodeContext` 对象，代表一个**正在构建中的 ProseMirror 节点**。

- `this.nodes[0]` 是最外层的节点（比如 `doc`）。
- `this.nodes[this.open]` 是当前正在接收内容的节点（栈顶）。
- `this.open` 是指向栈顶的指针。

当解析器遇到一个 `<h1>` 标签时，它会：

1.  创建一个代表 `heading` 的 `NodeContext`。
2.  将这个 `NodeContext` **压入** `this.nodes` 栈。
3.  `this.open++`。
4.  然后开始解析 `<h1>` 的子节点，并将结果（如 `text` 节点）添加到新的栈顶 `NodeContext` 的 `content` 数组中。
5.  当 `<h1>` 解析完毕后，将栈顶的 `NodeContext` **弹出**，调用其 `finish()` 方法构建出最终的 `heading` 节点，并将其添加到下一层的 `NodeContext`（即 `doc`）的 `content` 中。

#### 2. `NodeContext` - 正在构建的节点

`NodeContext` 对象包含了构建一个节点所需的所有信息：

- `type`, `attrs`, `marks`: 节点的类型、属性和应用的标记。
- `content: Node[]`: 已经解析完成并添加到此节点的子节点数组。
- `match: ContentMatch`: **至关重要**。这是一个 `ContentMatch` 实例，它跟踪着当前已添加的 `content` 是否仍然符合 `type` 的内容规则。每当要添加一个新子节点时，解析器都会用 `match` 来检查是否合法。

#### 3. 核心流程：`addDOM` -> `addElement` / `addTextNode`

解析的入口是 `addAll`，它遍历 DOM 子节点，并为每个节点调用 `addDOM`。

- `addDOM(dom)`: 判断 DOM 节点类型。

  - 如果是文本节点 (`nodeType == 3`)，调用 `addTextNode`。
  - 如果是元素节点 (`nodeType == 1`)，调用 `addElement`。

- `addTextNode(dom)`:

  - 处理空白符（根据 `preserveWhitespace` 选项）。
  - 创建一个 ProseMirror `TextNode`。
  - 调用 `insertNode` 将其插入到当前上下文中。

- `addElement(dom)`:

  1.  查找匹配此 `dom` 节点的 `ParseRule` (`this.parser.matchTag`)。
  2.  **如果没有规则匹配**，或者规则是 `skip`：则“透传”，直接递归解析其子节点 (`this.addAll(dom, ...)`）。如果它是一个块级标签，会尝试关闭当前的内联上下文（`sync` 逻辑）。
  3.  **如果规则匹配 `ignore`**: 则忽略。
  4.  **如果规则匹配 `node` 或 `mark`**: 调用 `addElementByRule`。

- `addElementByRule(dom, rule)`:
  - 如果规则是 `node`，则调用 `this.enter()` 创建并压入一个新的 `NodeContext`，然后递归解析其内容。
  - 如果规则是 `mark`，则将该 `mark` 添加到当前激活的标记集合中，然后递归解析其内容。

#### 4. 智能适配：`findPlace` 和 `findWrapping`

这是 `ParseContext` 最智能、最神奇的部分，用于处理不规范的 HTML。

当 `insertNode` 尝试插入一个节点（例如，一个 `list_item`）时，它发现当前的上下文（`this.top`，可能是一个 `paragraph`）不允许直接包含 `list_item`。这时会发生什么？

1.  `insertNode` 调用 `findPlace(node)`。
2.  `findPlace` 从当前深度开始，向上遍历 `this.nodes` 栈。
3.  在每一层，它询问 `NodeContext` 的 `match` 属性（通过 `cx.findWrapping(node)`）：“为了让你能合法地包含这个 `node`，我需要自动插入哪些包装节点？”
4.  `ContentMatch.findWrapping` 会返回一个节点类型数组，例如 `[ul]`。
5.  `findPlace` 找到了一个可行的“路线”：关闭当前的 `paragraph`，然后自动创建一个 `ul`，再把 `list_item` 放进去。
6.  它通过 `this.sync(sync!)` 将 `nodes` 栈回滚到需要插入包装节点的层级，然后通过 `this.enterInner` 压入自动创建的包装节点（`ul`），最后成功插入 `list_item`。

正是这个机制，使得 ProseMirror 能够将一段独立的 `<li>...</li>` HTML 片段，智能地解析成 `<ul><li>...</li></ul>`。

### 总结

from_dom.ts 是一个设计极其精巧的解析引擎，它完美地结合了**规则驱动**的灵活性和**状态机**的严谨性。

1.  **`ParseRule` 是声明式的蓝图**：它让用户能够以一种清晰的方式定义 DOM 和 ProseMirror 模型之间的映射关系。
2.  **`ParseContext` 是核心状态机**：它通过维护一个 `NodeContext` 栈，模拟了 ProseMirror 节点的构建过程。
3.  **`ContentMatch` 是 Schema 的守护者**：在 `NodeContext` 中，`ContentMatch` 实例在解析的每一步都强制执行 Schema 规则。
4.  **`findPlace` 和 `findWrapping` 是智能修复器**：它们是处理不规范 HTML 的关键，能够自动插入缺失的包装节点，使解析结果始终符合 Schema。

虽然代码量巨大且逻辑复杂，但其核心思想——**一个带智能修复功能的、基于 Schema 规则的递归下降解析器**——是清晰而强大的。掌握了它，你就掌握了 ProseMirror 如何驯服混乱的 Web 世界，并将其纳入自己严谨的模型体系之中的奥秘。
