好的，我们继续深入。在理解了 `Schema` 如何定义文档的“规则”之后，现在我们来看 node.ts，它定义了文档的“实体”——`Node` 类。这是构成 ProseMirror 文档的实际数据结构。

如果说 `Schema` 是宪法，那么 `Node` 就是遵循宪法构建的国家中的一砖一瓦。

### 核心哲学：不可变性 (Immutability)

在分析代码之前，你必须理解 `Node` 的核心设计哲学：**`Node` 是一个不可变（immutable）或持久化（persistent）的数据结构。**

这意味着你**永远不会去修改一个已存在的 `Node` 实例**。当你需要改变文档时（比如输入文字、删除段落），你实际上是创建了一个包含变化的新 `Node`（通常是新的顶层 `doc` 节点）。旧的 `Node` 实例保持不变。

这种设计有几个巨大的优势：

1.  **可靠的撤销/重做**: 历史记录只需要保存一个指向旧 `Node`（旧的文档状态）的引用即可，实现起来既简单又高效。
2.  **协同编辑**: 比较不同版本的文档状态变得非常直接，这是实现协同算法的基础。
3.  **UI 渲染**: 在与 React、Vue 等现代 UI 框架集成时，可以通过简单的引用比较 (`oldDoc === newDoc`) 来快速判断状态是否改变，从而决定是否需要重新渲染。
4.  **结构共享**: 创建新版本时，只有发生变化的路径上的节点需要被重新创建。文档中未改变的部分（比如其他段落）可以直接被新旧两个版本的文档树共享，这使得更新操作的内存和性能开销非常低。

现在，让我们带着“不可变性”这个核心思想来剖析 node.ts。

---

### `Node` 类的源码分析

`Node` 类代表了文档树中的一个节点。整个文档本身就是一个 `Node`，它的子节点也是 `Node`。

#### 1. 构造函数与核心属性

```typescript
// ...existing code...
export class Node {
  constructor(
    readonly type: NodeType,
    readonly attrs: Attrs,
    content?: Fragment | null,
    readonly marks = Mark.none
  ) {
    this.content = content || Fragment.empty
  }
// ...existing code...
```

一个 `Node` 实例由四个不可变的属性定义其身份：

- `type: NodeType`: 指向 `Schema` 中定义的节点类型。它决定了这个节点的规则，比如它是什么（段落、标题），允许包含什么内容等。
- `attrs: Attrs`: 一个包含此节点属性的对象，如 `image` 节点的 `src` 或 `alt`。
- `content: Fragment`: **这是关键**。`content` 是一个 `Fragment` 实例，`Fragment` 本质上是一个 `Node` 数组的封装。它代表了此节点的**所有子节点**。对于叶子节点（如 `image`），`content` 是 `Fragment.empty`。
- `marks: readonly Mark[]`: 应用于此节点的标记数组。通常只对内联节点有意义。

#### 2. 尺寸与索引系统 (`nodeSize`)

```typescript
// ...existing code...
  get nodeSize(): number {
    return this.isLeaf ? 1 : 2 + this.content.size
  }
// ...existing code...
```

`nodeSize` 是 ProseMirror 定位系统的基石。它定义了一个节点在文档的“扁平位置”模型中占据多少空间。

- **叶子节点 (Leaf Node)**: 如 `image` 或 `horizontal_rule`，尺寸为 `1`。
- **文本节点 (`TextNode`)**: 它的尺寸是其字符串长度（`text.length`）。
- **非叶子节点 (Non-leaf Node)**: 如 `paragraph` 或 `blockquote`，它的尺寸是 `2 + this.content.size`。这里的 `content.size` 是其所有子节点 `nodeSize` 的总和。那 `+2` 是什么呢？你可以把它们想象成节点的“开放标签”和“闭合标签”。

例如，一个包含文本 "Hi" 的段落 `<p>Hi</p>`，其结构是 `paragraph(text("Hi"))`。

- `text("Hi")` 的 `nodeSize` 是 2。
- `paragraph` 的 `nodeSize` 是 `2 + text.nodeSize` = `2 + 2` = `4`。

这套索引系统使得文档中的任意位置都可以用一个唯一的整数来表示。

#### 3. 子节点访问与遍历

`Node` 上的很多方法其实是其 `content` 属性（一个 `Fragment` 实例）对应方法的**代理**。

- `child(index)`, `maybeChild(index)`, `childCount`, `firstChild`, `lastChild`: 提供对直接子节点的访问。
- `forEach(f)`: 遍历直接子节点。
- `nodesBetween(from, to, f)`: **非常重要**。递归地遍历指定范围内的所有**后代**节点。这是许多操作（如查找、应用样式）的基础。
- `descendants(f)`: `nodesBetween` 的一个特例，遍历所有后代节点。

#### 4. 不可变性操作：创建新节点

这些方法完美地体现了不可变性哲学。它们从不修改当前节点，而是返回一个**新的 `Node` 实例**。

- `copy(content)`: 创建一个与当前节点拥有相同 `type`, `attrs`, `marks` 的新节点，但内容替换为给定的 `content`。如果 `content` 与当前 `content` 相同，它会直接返回 `this`，这是一个常见的优化。
- `mark(marks)`: 创建一个内容、类型、属性都相同，但 `marks` 不同的新节点。
- `cut(from, to)`: 创建一个新节点，其内容是当前节点内容的一个片段。

#### 5. 比较与查询

- `eq(other)`: 深度比较两个节点是否完全相同。它首先调用 `sameMarkup`，然后递归地调用 `content.eq`。
- `sameMarkup(other)`: 仅比较节点的“外壳”（`type`, `attrs`, `marks`）是否相同，不关心内容。这里用到了我们之前分析的 `compareDeep` 来比较 `attrs`。
- `hasMarkup(...)`: `sameMarkup` 的底层实现。

#### 6. 核心修改操作：`slice` 和 `replace`

- `slice(from, to)`: 从节点中“切”出一个 `Slice` 对象。`Slice` 不仅仅是内容的片段，它还记录了切片“开放”的深度。例如，如果你从一个段落中间开始切，到另一个段落中间结束，那么这个 `Slice` 的两端就是开放的，它知道自己需要被包裹在 `paragraph` 节点里。这是实现复制/粘贴的关键。
- `replace(from, to, slice)`: **这是整个模型中最重要的修改方法**。它用一个 `Slice` 替换节点内从 `from` 到 `to` 的内容。ProseMirror 中几乎所有的文档修改操作（输入、删除、粘贴等）最终都会归结为一次 `replace` 调用。这个方法的复杂逻辑被委托给了 replace.ts 中的 `replace` 函数。

#### 7. Schema 验证

- `canReplace(from, to, replacement)`: 在执行 `replace` 之前，可以用这个方法来检查替换是否符合 `Schema` 规则。它利用 `NodeType` 上的 `contentMatch` 来进行高效的验证。
- `check()`: 递归地检查整个节点树是否完全符合 `Schema` 规则，包括节点内容、属性、标记等。这是一个用于调试和保证数据完整性的强大工具。

#### 8. 序列化

- `toJSON()`: 将节点转换为一个可被 `JSON.stringify` 的普通 JavaScript 对象。
- `static fromJSON(schema, json)`: 从 JSON 对象反序列化出一个 `Node` 实例。注意它需要一个 `schema` 参数，因为它需要通过 `schema` 来查找 `NodeType` 并创建节点。

---

### `TextNode` 子类

`TextNode` 是 `Node` 的一个特殊子类，专门用于表示文本。

- 它没有 `content`（子节点），而是有一个 `text: string` 属性来存储文本内容。
- 它的 `nodeSize` 直接就是 `this.text.length`。
- 它重写了 `cut`, `eq` 等方法，以直接操作 `text` 字符串，而不是 `content`。
- 构造函数会检查并禁止创建空的 `TextNode`，这是 ProseMirror 的一个重要约束。

### 总结

`Node` 类是 ProseMirror 数据模型的具体实现。它通过**不可变性**和**结构共享**，提供了一个既健壮又高效的文档表示方式。它与 `Schema` 和 `NodeType` 紧密协作，`NodeType` 定义了“是什么”和“应该怎样”，而 `Node` 则是“就是这个”的实际数据。

理解 `Node` 的 `nodeSize`、不可变操作 (`copy`, `mark`, `cut`) 以及核心的 `replace` 方法，是掌握 ProseMirror 文档操作的关键。
