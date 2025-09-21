好的，我们来攻克 resolvedpos.ts。这是 prosemirror-model 中一个极其重要的概念。如果说 `Node` 和 `Fragment` 是文档的“静态解剖图”，那么 `ResolvedPos` 就是一个“动态的 GPS 系统”，它能告诉你关于文档中任意一个点的所有信息。

### 宏观理解：为什么需要 `ResolvedPos`？

在 ProseMirror 中，文档中的一个位置（position）只是一个简单的整数。例如，在文档 `<p>Hello</p>` 中，位置 `3` 就在 "H" 和 "e" 之间。

但一个孤零零的数字 `3` 是“愚蠢的”，它本身不包含任何上下文。我们经常需要回答以下问题：

- 位置 `3` 在哪个段落里？在哪个文档里？
- 它在父节点（段落）中的偏移量是多少？
- 它前面和后面的节点是什么？
- 在位置 `3` 输入文字时，应该继承哪些标记（比如粗体、链接）？
- 从位置 `3` 到位置 `10` 是否在同一个文本块内？

`ResolvedPos` (通常简写为 `$pos`) 就是为了回答这些问题而生的。它接收一个“愚蠢”的整数 `pos` 和一个 `doc` 节点，然后返回一个包含极其丰富的上下文信息的**对象**。ProseMirror 中几乎所有复杂的命令、插件和 UI 逻辑都严重依赖 `ResolvedPos`。

---

### 核心内部结构：`path` 数组

`ResolvedPos` 的所有魔力都源于其内部的一个属性：`path`。这个 `path` 数组是在 `ResolvedPos.resolve` 静态方法中被计算出来的。

`path` 数组存储了从文档根节点到目标位置所经过的所有节点的“面包屑”路径。它是一个扁平化的数组，每三个元素为一组，代表路径上的一层：

`[node_level_0, index_in_level_0, offset_of_level_0, node_level_1, index_in_level_1, offset_of_level_1, ...]`

- `node`: 路径上这一层的父节点。
- `index`: 在这个父节点中，路径走向了它的第几个子节点。
- `offset`: 这个父节点的内容区域（在整个文档中）的起始位置。

**示例**: 对于文档 `doc(paragraph(text("Hi")))` 和位置 `2` (在 "H" 和 "i" 之间):

- `path` 数组会是这样的：
  - `[docNode, 0, 0,` (在 `doc` 节点的第 `0` 个子节点，`doc` 的内容区从 `0` 开始)
  - `paragraphNode, 0, 1,` (在 `paragraph` 节点的第 `0` 个子节点，`paragraph` 的内容区从 `1` 开始)
  - `textNode, 2, 2]` (在 `text` 节点中，偏移量为 `2`，文本节点的起始位置是 `2`)
- `depth`: 路径的深度，等于 `path.length / 3 - 1`。

有了这个 `path` 数组，`ResolvedPos` 就可以通过简单的数组索引和计算，光速回答各种关于此位置的上下文问题。

---

### `ResolvedPos` 类的源码分析

#### 1. 静态工厂方法：`resolve` 和 `resolveCached`

这是创建 `ResolvedPos` 实例的唯一入口。

- `static resolve(doc, pos)`: 这是核心的计算函数。它从 `doc` 节点开始，根据 `pos` 一层层地向下遍历，使用 `node.content.findIndex()` 找到在每一层应该进入哪个子节点，以及在该子节点内的相对偏移量。它一边遍历，一边构建我们上面讲的 `path` 数组。
- `static resolveCached(doc, pos)`: **重要的性能优化**。解析位置是一个非常频繁的操作。这个方法使用一个 `WeakMap` 来为每个 `doc` 实例缓存最近解析过的几个 `ResolvedPos` 对象。如果请求的位置在缓存中，就直接返回，避免了重复的、昂贵的 `resolve` 计算。

#### 2. 核心属性和访问器

一旦 `ResolvedPos` 被创建，它就提供了大量便捷的属性和方法：

- `pos`: 原始的整数位置。
- `depth`: 当前位置的深度。`0` 表示在 `doc` 的顶层。
- `parent`: 当前位置所在的直接父节点。等价于 `this.node(this.depth)`。
- `doc`: 文档的根节点。等价于 `this.node(0)`。
- `parentOffset`: 当前位置相对于其 `parent` 内容区起始点的偏移量。

#### 3. 祖先信息 (`node`, `index`, `start`, `end`, `before`, `after`)

这些方法允许你查询路径上任意一层的祖先节点的信息。它们都接受一个可选的 `depth` 参数。

- `node(depth)`: 获取指定深度的祖先节点。
- `index(depth)`: 获取在指定深度的祖先节点中，路径走向了第几个子节点。
- `start(depth)`: 获取指定深度祖先节点**内容区**的起始位置。
- `end(depth)`: 获取指定深度祖先节点**内容区**的结束位置。
- `before(depth)`: 获取指定深度祖先节点**自身**的起始位置（在它的父节点中的位置）。
- `after(depth)`: 获取指定深度祖先节点**自身**的结束位置。

**示例**: 对于 `<p>Hello</p>` (假设它在 `doc` 的顶层)，一个指向 "e" 的 `$pos`：

- `$pos.parent` 是 `paragraph` 节点。
- `$pos.start()` 是段落内容区的开始位置（`<p>` 之后）。
- `$pos.end()` 是段落内容区的结束位置（`</p>` 之前）。
- `$pos.before()` 是段落节点自身的开始位置（`<p>` 之前）。
- `$pos.after()` 是段落节点自身的结束位置（`</p>` 之后）。

#### 4. 相邻节点信息 (`nodeBefore`, `nodeAfter`, `textOffset`)

- `textOffset`: 如果位置在一个文本节点内，返回它在文本节点中的偏移量（从 0 开始）。否则为 0。
- `nodeBefore`: 返回紧邻在当前位置之前的节点。
- `nodeAfter`: 返回紧邻在当前位置之后的节点。

#### 5. 标记信息 (`marks`, `marksAcross`)

这是 `ResolvedPos` 最强大的功能之一，对实现格式刷、保持光标样式等至关重要。

- `marks()`: 计算并返回在**当前位置**应该生效的标记集合。它的逻辑很智能：
  - 如果在一个文本节点中间，直接返回该文本节点的标记。
  - 如果在两个节点之间，它会查看两边节点的标记，并根据标记的 `inclusive` 属性来决定。一个非 `inclusive` 的标记（如 `link`）只在它所应用的节点的内部生效，而不会“延伸”到边界之外。`marks()` 方法会正确处理这种情况，返回一个在光标处输入时应该被应用的、最符合用户直觉的标记集合。
- `marksAcross($end)`: 当你删除一个范围的文本时，你需要知道删除后新连接起来的文本应该是什么样式。这个方法就是用来计算这个的。它会查看范围终点 `$end` 的标记，并与当前位置的标记进行比较，以决定哪些标记可以在删除操作后“存活”下来。

#### 6. 范围与比较 (`sharedDepth`, `blockRange`, `sameParent`)

- `sharedDepth(pos)`: 计算当前位置和另一个位置共同拥有的祖先节点的深度。
- `blockRange(other)`: 一个非常实用的方法。它会查找当前位置和另一个位置 `other` 之间所共有的、最内层的**块级节点**范围。例如，如果两个位置都在同一个段落里，它会返回一个覆盖该段落的 `NodeRange`。这是许多基于块的命令（如设置段落样式、提升/降低列表项）的基础。
- `sameParent(other)`: 快速判断两个位置是否在同一个父节点中。

---

### `NodeRange` 类

这个文件也定义了一个小的辅助类 `NodeRange`。它代表一个“扁平”的范围，即在一个父节点内，从 `startIndex` 到 `endIndex` 的所有相邻子节点。它由 `blockRange` 等方法返回，并提供了方便的 `start`, `end`, `parent`, `startIndex`, `endIndex` 属性。

### 总结

`ResolvedPos` 是 ProseMirror 模型层中连接“数据”和“逻辑”的桥梁。它将一个简单的整数位置，解码成一个包含深度、父子关系、兄弟节点、标记上下文等丰富信息的对象。

- **核心是 `path` 数组**：一个预计算的、从根到目标位置的“面包屑”路径。
- **性能是关键**：通过 `resolveCached` 实现高效的位置解析。
- **API 是核心价值**：提供了一整套强大、便捷的方法来查询位置的上下文，极大地简化了上层（如 prosemirror-state 和 prosemirror-commands）的逻辑复杂度。

掌握了 `ResolvedPos`，你就拥有了在 ProseMirror 文档中“导航”和“感知”的超能力，这是编写任何高级功能的先决条件。
