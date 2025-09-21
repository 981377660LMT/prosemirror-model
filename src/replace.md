好的，我们终于来到了 prosemirror-model 的“圣杯”——`replace.ts`。这是整个模型包中功能最强大、逻辑最复杂、也最核心的部分。ProseMirror 中**所有**的文档修改操作，无论是输入一个字符、删除一个段落，还是粘贴一段复杂的 HTML，最终都会被转换成一次对 `replace` 函数的调用。

理解了 replace.ts，你就理解了 ProseMirror 是如何以一种原子性的、符合 Schema 规则的方式，来执行任意复杂的文档变换的。

### 宏观理解：`replace` 的使命

`replace` 函数的目标是：**用一个 `Slice` 对象，替换文档中从 `$from` 到 `$to` 的内容。**

这个过程远比听起来要复杂，因为它必须：

1.  **尊重 Schema**：最终产生的节点必须完全符合其父节点的 `contentMatch` 规则。
2.  **智能地“缝合” (Stitching)**：它需要智能地处理替换区域边界的情况。例如，如果将一段文本粘贴到一个段落的中间，它应该保持在同一个段落里。如果将一个列表项粘贴到另一个列表项的中间，它应该成为一个新的列表项，而不是破坏列表结构。
3.  **保持不可变性**：操作的结果是一个全新的 `doc` 节点，原节点保持不变。

为了完成这个复杂的任务，`replace.ts` 引入了一个至关重要的概念：`Slice`。

---

### 第一部分：`Slice` - 超级剪贴板

`Slice` 是 ProseMirror 的“剪贴板”数据结构。它比 `Fragment` 更强大，因为它不仅存储了**内容**，还存储了内容被“切开”时的**上下文**。

#### 1. `openStart` 和 `openEnd`

这是 `Slice` 的核心。这两个数字代表了 `Slice` 内容的左侧和右侧被“切开”的深度。

**用一个具体的例子来理解：**
假设我们有如下文档结构：
`<blockquote><p>one</p><p>two</p></blockquote>`

- **场景 A：复制 "one"**

  - 你选中的是 `text("one")`。
  - `slice.content` 是 `Fragment(text("one"))`。
  - 切片的左右两边都没有切开任何父节点。
  - `openStart = 0`, `openEnd = 0`。

- **场景 B：复制 `</p><p>` (即从第一个段落末尾到第二个段落开头)**

  - 你选中的是 `paragraph(text("one"))` 和 `paragraph(text("two"))`。
  - `slice.content` 是 `Fragment(paragraph(text("one")), paragraph(text("two")))`。
  - 切片的左侧是在 `blockquote` 内部，但**没有**切开 `paragraph`。所以左侧是“闭合”的。
  - 切片的右侧也是在 `blockquote` 内部，也是“闭合”的。
  - `openStart = 1`, `openEnd = 1`。这里的 `1` 代表 `blockquote` 这一层是开放的。

- **场景 C：复制 `one</p><p>tw`**
  - 你选中的内容跨越了两个段落的内部。
  - `slice.content` 是 `Fragment(paragraph(text("one")), paragraph(text("tw")))`。
  - 切片的左侧切开了 `blockquote` **和** 第一个 `paragraph`。
  - 切片的右侧切开了 `blockquote` **和** 第二个 `paragraph`。
  - `openStart = 2`, `openEnd = 2`。

`openStart` 和 `openEnd` 告诉 `replace` 算法：“我这块内容是从一个深度为 N 的结构里挖出来的，当你粘贴我的时候，请尝试把我‘缝合’回一个同样深度为 N 的结构里去。”

#### 2. `Slice.size`

`get size(): number { return this.content.size - this.openStart - this.openEnd; }`
这个计算很有趣。它返回的是**插入后文档实际增加的尺寸**。`openStart` 和 `openEnd` 的部分代表了要被“吸收”或“合并”的节点标签，所以要从总尺寸中减去。

---

### 第二部分：`replace` 算法 - 递归的艺术

`replace` 算法的核心思想是一个**从上到下、分而治之的递归过程**。

#### 1. 入口：`replace($from, $to, slice)`

这是公共 API。它首先做一些健全性检查，确保 `slice` 的开放深度与插入/删除位置的深度是兼容的，然后调用 `replaceOuter` 启动递归。

#### 2. 核心递归：`replaceOuter`

`replaceOuter` 在指定的 `depth` 级别上工作。它会判断在当前 `depth` 下，替换操作属于哪种情况：

- **情况一：深入递归**

  - 如果 `$from` 和 `$to` 在当前 `depth` 下指向同一个子节点，并且替换操作还没有到达 `slice` 需要被“打开”的深度，那么问题就被简化为：“在更深一层的子节点内部进行替换”。
  - 它会递归调用 `replaceOuter(..., depth + 1)`，然后用返回的新子节点替换旧的子节点。

- **情况二：简单删除**

  - 如果 `slice` 的内容为空（即纯删除操作），则调用 `replaceTwoWay`。

- **情况三：简单、扁平的替换**

  - 如果 `slice` 是完全闭合的 (`openStart` 和 `openEnd` 都为 0)，并且替换范围 `$from` 和 `$to` 也在同一个父节点内，那么这就是最简单的情况。直接用 `slice.content` 替换父节点内容的一部分即可。

- **情况四：最复杂的“三路合并”**
  - 这是最复杂也是最常见的情况，涉及到处理开放的 `slice`。它会调用 `replaceThreeWay`。

#### 3. 替换策略：`replaceTwoWay` 和 `replaceThreeWay`

这两个函数是实际执行内容拼接的地方。它们的名字描述了它们如何处理边界：

- `replaceTwoWay($from, $to, depth)`: 用于**纯删除**。它构建一个新的 `Fragment`，包含 `$from` 左侧的内容和 `$to` 右侧的内容。在 `$from` 和 `$to` 的连接点，它会递归调用 `replaceTwoWay` 来尝试“缝合”更深层次的节点。

- `replaceThreeWay($from, $start, $end, $to, depth)`: 用于**带内容的替换**。它构建一个新的 `Fragment`，由三部分组成：

  1.  `$from` 左侧的内容。
  2.  `slice` 的内容（已经被 `prepareSliceForReplace` 包装并解析出了 `$start` 和 `$end`）。
  3.  `$to` 右侧的内容。

  在 `1` 和 `2` 的连接处，以及 `2` 和 `3` 的连接处，它都会检查节点是否可以“连接”（`joinable`），如果可以，就会递归地调用 `replaceTwoWay` 或 `replaceThreeWay` 来缝合更深层次的节点。

#### 4. 辅助工具

- `prepareSliceForReplace(slice, $along)`: 这是一个非常关键的预处理步骤。它接收要插入的 `slice`，并根据目标位置 `$along` 的上下文，将 `slice` 的内容包装在与目标位置相同类型的父节点中，直到达到 `slice` 的 `openStart` 深度。这就像是为了让一个插头能插进插座，先给它配好一个正确形状的转接头。
- `close(node, content)`: 在递归的每一层返回时，用这个函数来确保新生成的内容 `content` 符合父节点 `node` 的 `checkContent` 规则，然后创建一个新的 `node` 副本。
- `joinable(..., depth)`: 检查在指定深度，两个位置的父节点是否是兼容的，可以被“缝合”在一起。
- `addRange` / `addNode`: 用于构建新 `Fragment` 内容的辅助函数，`addNode` 会负责合并相邻的文本节点。

### 总结

replace.ts 是 ProseMirror 模型层的心脏。它通过一个优雅但复杂的递归算法，实现了对不可变文档树的原子性、符合 Schema 的修改。

1.  **`Slice` 是关键数据结构**：它通过 `openStart` 和 `openEnd` 携带了复制/剪切操作的上下文，指导 `replace` 算法如何进行智能“缝合”。
2.  **递归是核心思想**：算法通过 `replaceOuter` -> `replaceThreeWay`/`TwoWay` 的递归调用，将复杂的、跨越多层级的替换操作，分解为在每一层级上对节点序列的拼接问题。
3.  **“缝合”是核心操作**：在每一层递归中，算法都会尝试在替换边界上 `joinable`（连接）节点，从而实现 `<p>ab</p>` + `<p>cd</p>` -> `<p>abcd</p>` 这样的智能合并。
4.  **Schema 是最高准则**：在递归的每一层返回时，`close` 函数都会调用 `checkContent`，确保所有操作的结果都严格遵守 Schema 规则，否则就会抛出 `ReplaceError`。

虽然 replace.ts 的代码初看起来令人望而生畏，但理解了其核心的递归思想和 `Slice` 的作用后，你就能领会到 ProseMirror 数据模型设计的强大与严谨。
