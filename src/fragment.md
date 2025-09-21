好的，我们来攻克 fragment.ts。这是理解 ProseMirror 数据模型的关键一步，因为它揭示了 `Node` 内部 `content` 的工作机制。

### 宏观理解：`Fragment` 是什么？为什么需要它？

`Fragment` 代表一个**有序的、不可变的节点序列**。简单来说，它就是 `Node` 的 `content`（子节点列表）的容器。

你可能会问：**为什么不直接用一个 JavaScript 数组 `Node[]` 呢？**

这是一个绝佳的问题，答案揭示了 ProseMirror 的设计精髓：

1.  **强制不可变性 (Immutability)**: JavaScript 的数组是可变的 (`push`, `splice` 等会修改原数组)。`Fragment` 作为一个类，其所有“修改”方法（如 `cut`, `append`）都返回一个**新的 `Fragment` 实例**，而原实例保持不变。这与 `Node` 的不可变哲学完全一致，保证了数据模型的一致性和可预测性。
2.  **API 的丰富与一致性**: `Fragment` 提供了一套专门为操作节点序列而设计的、高度优化的 API（如 `nodesBetween`, `cut`, `findDiffStart`）。这些 API 与 `Node` 上的方法相呼应，提供了统一的编程体验。
3.  **性能优化**:
    - **缓存 `size`**: `Fragment` 实例会缓存其所有子节点 `nodeSize` 的总和。对于一个大数组，每次都重新计算总尺寸是低效的。
    - **单例 `Fragment.empty`**: 所有叶子节点（没有子节点的节点）都共享同一个空的 `Fragment` 实例。这避免了为成千上万的叶子节点重复创建无数个空数组/空 Fragment，极大地节省了内存。
4.  **规范化 (Normalization)**: `Fragment` 的工厂方法（如 `fromArray`）会自动执行一个重要的规范化操作：**合并相邻的、具有相同标记的文本节点**。这确保了文档模型始终处于最简洁、最规范的状态。

现在，让我们深入代码。

---

### `Fragment` 类的源码分析

#### 1. 构造函数与核心属性

```typescript
export class Fragment {
  readonly size: number

  constructor(
    readonly content: readonly Node[],
    size?: number
  ) {
    this.size = size || 0
    if (size == null) for (let i = 0; i < content.length; i++) this.size += content[i].nodeSize
  }
// ...
```

- `content: readonly Node[]`: 存储节点序列的内部数组。`readonly` 关键字提示我们不应该直接修改它。
- `size: number`: 缓存的、所有子节点 `nodeSize` 的总和。构造函数逻辑很清晰：如果传入了 `size` 就直接使用（性能优化，因为调用者通常已经知道尺寸），否则就遍历 `content` 数组计算出来。

#### 2. 静态工厂方法：如何创建 `Fragment`

这是获取 `Fragment` 实例的主要入口。

- `static empty`: 一个预先创建好的、代表空的 `Fragment` 的**单例**。这是重要的性能优化。
- `static from(nodes?)`: 这是一个非常方便的、多态的工厂方法。
  - `null` 或 `undefined` -> `Fragment.empty`
  - 传入一个 `Fragment` -> 直接返回它自己
  - 传入一个 `Node` -> 创建一个只包含该节点的 `Fragment`
  - 传入一个 `Node[]` -> 调用 `fromArray` 处理
- `static fromArray(array)`: 这是核心。它接收一个节点数组，并执行**文本节点合并**的规范化操作。
  ```typescript
  // ...
  if (i && node.isText && array[i - 1].sameMarkup(node)) {
    // ...
    joined[joined.length - 1] = (node as TextNode).withText(
      (joined[joined.length - 1] as TextNode).text + (node as TextNode).text
    );
  // ...
  ```
  它遍历数组，如果发现当前节点是一个文本节点，并且它前一个节点也是具有相同标记（`sameMarkup`）的文本节点，它就会将两个文本节点合并成一个。这保证了文档中不会出现 `text("a"), text("b")` 这样的冗余结构，而是会自动变成 `text("ab")`。

#### 3. 不可变“修改”方法 (你高亮的部分)

这些方法是 `Fragment` 不可变性的具体体现。

- `append(other)`: 将两个 `Fragment` 连接成一个新的 `Fragment`。

  ```typescript
  // filepath: /Users/bytedance/coding/pm/prosemirror-model/src/fragment.ts
  // ...existing code...
  let last = this.lastChild!,
    first = other.firstChild!,
    content = this.content.slice(), // 创建一个新数组
    i = 0
  if (last.isText && last.sameMarkup(first)) {
    // 优化：合并连接处的文本节点
    content[content.length - 1] = (last as TextNode).withText(last.text! + first.text!)
    i = 1
  }
  for (; i < other.content.length; i++) content.push(other.content[i])
  return new Fragment(content, this.size + other.size) // 返回新实例
  // ...existing code...
  ```

  注意，它也做了文本节点合并的优化：如果第一个 `Fragment` 的结尾和第二个 `Fragment` 的开头是可合并的文本节点，它会先将它们合并。

- `cut(from, to)`: **基于位置**截取一个子 `Fragment`。这是 `Node.cut` 的底层实现。

  ```typescript
  // filepath: /Users/bytedance/coding/pm/prosemirror-model/src/fragment.ts
  // ...existing code...
  if (pos < from || end > to) {
    if (child.isText)
      child = child.cut(...) // 递归调用 TextNode.cut
    else
      child = child.cut(...) // 递归调用 Node.cut
  }
  result.push(child)
  // ...existing code...
  ```

  它的逻辑很精巧：遍历子节点，计算每个节点的起止位置。

  - 完全在 `[from, to]` 范围内的子节点，直接放入 `result` 数组。
  - 部分与 `[from, to]` 范围相交的子节点（即切片的起点或终点在该节点内部），则**递归调用 `child.cut()`**，只取其需要的部分。这体现了数据结构操作的递归本质。

- `replaceChild(index, node)`: **基于索引**替换一个子节点。
  ```typescript
  // filepath: /Users/bytedance/coding/pm/prosemirror-model/src/fragment.ts
  // ...existing code...
  let current = this.content[index]
  if (current == node) return this // 优化：如果没变化，返回自身
  let copy = this.content.slice() // 关键：创建数组副本
  let size = this.size + node.nodeSize - current.nodeSize
  copy[index] = node
  return new Fragment(copy, size) // 返回新实例
  // ...existing code...
  ```
  这个方法完美地展示了不可变性和结构共享：它只创建了一个新的顶层数组，但数组中未被替换的 `Node` 元素仍然是旧的引用，被新旧两个 `Fragment` 共享。

#### 4. 遍历与查询方法

- `nodesBetween(from, to, f)`: 与 `Node.nodesBetween` 类似，递归遍历指定范围内的所有后代节点。这是实现各种文档分析功能（如提取文本、查找内容）的基石。
- `descendants(f)`: `nodesBetween` 的特例，遍历所有后代。
- `textBetween(...)`: 使用 `nodesBetween` 提取指定范围内的纯文本内容。
- `findDiffStart(other)` / `findDiffEnd(other)`: **非常重要的 diff 算法**。它们用于高效地比较两个 `Fragment`，并找出它们开始和结束发生差异的位置。这是 ProseMirror 视图层（prosemirror-view）进行高效 DOM 更新的核心，也是协同编辑中计算变更集的基础。

#### 5. 辅助方法

- `child(index)`, `maybeChild(index)`, `childCount`, `firstChild`, `lastChild`: 提供了方便的、基于索引的子节点访问器。
- `eq(other)`: 深度比较两个 `Fragment` 的内容是否完全相等。
- `toJSON()` / `static fromJSON(schema, value)`: 实现与 JSON 格式的相互转换。

### 总结

`Fragment` 是一个看似简单但设计极其精巧的类。它不仅仅是一个数组包装器，更是 ProseMirror 不可变数据模型哲学的重要组成部分。

- 它通过**强制不可变性**和提供一套丰富的 API，保证了数据操作的安全性和一致性。
- 它通过**文本节点合并**实现了模型的规范化。
- 它通过**缓存 `size`** 和**共享 `Fragment.empty`** 实现了性能优化。
- 它提供的 `findDiffStart/End` 方法是更上层模块实现高效 diff/patch 算法的基石。

理解了 `Fragment`，你就真正理解了 `Node` 的 `content` 是如何被管理和操作的，也为理解 ProseMirror 更复杂的文档变换（`replace`）和视图更新铺平了道路。
