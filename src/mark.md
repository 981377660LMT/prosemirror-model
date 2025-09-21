好的，我们来全面、深入地剖析 mark.ts。这个文件定义了 ProseMirror 文档模型中与“样式”和“元数据”相关的部分。我们已经了解了 `Node` 如何构建文档的结构，现在我们来看看 `Mark` 是如何为这些结构添加“色彩”的。

### 宏观理解：`Mark` 是什么？它与 `Node` 有何不同？

- **`Node` 定义结构**: `Node` 组成了文档的树状结构。一个 `paragraph` 包含 `text`，一个 `blockquote` 包含 `paragraph`。它们是文档的骨架。
- **`Mark` 定义信息**: `Mark` 将额外的信息附加到**内联内容**上，但**不改变其结构**。例如，将一段文字标记为**粗体** (`strong`) 或一个**链接** (`link`)。你可以把 `Mark` 想象成在文本上划的“高亮笔”，它不会改变文本本身，只是给它增加了一层含义。

一个 `Mark` 实例代表一个具体的标记，例如“一个指向 `https://prosemirror.net` 的链接”。它由两部分组成：

1.  `type: MarkType`: 标记的类型，如 `link`, `strong`, `em`。它来自 `Schema`。
2.  `attrs: Attrs`: 标记的属性，如 `link` 标记的 `href` 属性。

---

### 核心概念：`MarkSet` - 有序且不可变的标记集合

在 ProseMirror 中，一个节点（通常是 `TextNode`）可以同时拥有多个标记，例如一段既加粗又是链接的文本。这些标记被存储在一个集合中，这个集合在 ProseMirror 中被称为 `MarkSet`。

`MarkSet` 并不是一个独立的类，而是通过一个 `readonly Mark[]` 数组和一系列操作此数组的规则和方法来实现的。它有三个至关重要的特性：

1.  **不可变性 (Immutability)**: 与 `Node` 和 `Fragment` 一样，你永远不会修改一个 `MarkSet`。当你添加或删除一个标记时，会返回一个**新的** `MarkSet` 数组。
2.  **唯一性 (Uniqueness)**: 一个 `MarkSet` 中不会包含两个完全相同的 `Mark` 实例（`type` 和 `attrs` 都相同）。
3.  **有序性 (Sorted)**: `MarkSet` 中的 `Mark` 始终按照其 `type.rank`（在 `Schema` 中定义的顺序）进行排序。这提供了一个**规范表示**，使得比较两个 `MarkSet` 是否相等变得非常高效（只需逐个元素比较）。

现在，让我们带着这三个核心特性来分析 mark.ts 的代码。

---

### `Mark` 类的源码分析

#### 1. 构造函数与 `eq` 方法

```typescript
// ...existing code...
export class Mark {
  constructor(
    readonly type: MarkType,
    readonly attrs: Attrs
  ) {}

  // ...

  eq(other: Mark) {
    return this == other || (this.type == other.type && compareDeep(this.attrs, other.attrs))
  }
// ...existing code...
```

- `constructor`: 非常简单，只存储 `type` 和 `attrs`。
- `eq(other)`: 定义了两个 `Mark` 何时被认为是“相等”的。首先是快速路径检查（是否是同一个对象引用），然后检查 `type` 是否相同，并使用我们之前分析过的 `compareDeep` 来深度比较 `attrs` 对象。这是判断唯一性的基础。

#### 2. `addToSet(set)`: 核心的集合操作

这是 mark.ts 中最复杂也最重要的方法。它以一种符合所有规则（不可变、唯一、有序、互斥）的方式将当前 `Mark` 添加到一个 `MarkSet` 中。

让我们分解它的逻辑：

```typescript
// ...existing code...
  addToSet(set: readonly Mark[]): readonly Mark[] {
    let copy, placed = false;
    for (let i = 0; i < set.length; i++) {
      let other = set[i];
      // 1. 唯一性检查
      if (this.eq(other)) return set;

      // 2. 互斥规则 (Exclusion)
      if (this.type.excludes(other.type)) { // 新 mark 排除旧 mark
        if (!copy) copy = set.slice(0, i); // 开始复制，但不把 other 加入
      } else if (other.type.excludes(this.type)) { // 旧 mark 排除新 mark
        return set; // 新 mark 被拒绝，直接返回原 set
      } else {
        // 3. 排序规则 (Sorting)
        if (!placed && other.type.rank > this.type.rank) {
          if (!copy) copy = set.slice(0, i);
          copy.push(this); // 在 rank 更大的 mark 之前插入
          placed = true;
        }
        if (copy) copy.push(other);
      }
    }
    // 4. 收尾
    if (!copy) copy = set.slice(); // 如果循环中从未创建副本，现在创建
    if (!placed) copy.push(this); // 如果新 mark 的 rank 最大，在末尾添加
    return copy;
  }
// ...existing code...
```

- **`copy` 和 `placed` 标志**: 这是**惰性复制 (lazy copying)** 的实现。在遍历开始时，`copy` 是 `undefined`。只有当确定需要做出改变时（例如需要移除一个互斥标记，或需要插入当前标记），才会通过 `set.slice(0, i)` 创建一个新数组 `copy`。这避免了在不需要改变集合时（如标记已存在或被拒绝）创建新数组的开销。
- **1. 唯一性**: 如果要添加的 `Mark` 已经存在于 `set` 中，直接返回原 `set`。
- **2. 互斥规则**:
  - 如果新 `Mark` 的类型排斥 `set` 中某个现有 `Mark` 的类型（`this.type.excludes(other.type)`），那么这个现有的 `other` 将不会被加入到 `copy` 中，从而实现“替换”效果。
  - 反之，如果 `set` 中某个现有 `Mark` 排斥新 `Mark`，那么新 `Mark` 就不能被添加，直接返回原 `set`。
- **3. 排序规则**: 当遍历到一个 `other` 标记，其 `rank` 大于新 `Mark` 的 `rank` 时，就意味着找到了新 `Mark` 应该插入的位置（在 `other` 之前）。
- **4. 收尾**: 如果遍历完整个 `set` 都没有触发复制（`copy` 仍为 `undefined`），说明新 `Mark` 应该被添加到末尾，此时再创建副本。如果 `placed` 仍为 `false`，也说明新 `Mark` 的 `rank` 是最大的，需要添加到末尾。

#### 3. `removeFromSet(set)`

这个方法相对简单。它遍历 `set`，找到与 `this` 相等的 `Mark`，然后返回一个不包含该元素的新数组。如果找不到，直接返回原 `set`。这同样遵循了不可变性原则。

#### 4. 静态方法：`MarkSet` 的工厂和工具

这些静态方法提供了创建和管理 `MarkSet` 的便捷途径。

- `static none: readonly Mark[] = []`: 一个预先创建的、代表空集合的**单例**。与 `Fragment.empty` 一样，这是重要的性能优化。
- `static setFrom(marks?)`: 一个多态的工厂方法，用于从各种输入（`null`, 单个 `Mark`, `Mark` 数组）创建一个**规范的、有序的** `MarkSet`。它会调用 `sort` 来保证有序性。
- `static sameSet(a, b)`: 比较两个 `MarkSet` 是否相等。由于 `MarkSet` 是有序的，这个比较可以非常高效：首先检查引用和长度，然后逐个元素调用 `eq` 方法比较。**这就是为什么 `MarkSet` 必须排序的原因**。

#### 5. 序列化 (`toJSON`, `fromJSON`)

- `toJSON()`: 将 `Mark` 实例转换为可 JSON 序列化的普通对象，只包含 `type` 名称和 `attrs`。
- `static fromJSON(schema, json)`: 从 JSON 对象反序列化回一个 `Mark` 实例。它需要一个 `schema` 来查找对应的 `MarkType`，然后调用 `type.create()` 来创建实例。

### 总结

mark.ts 的设计精妙地解决了在不可变数据结构上管理一组带有关联规则（唯一、有序、互斥）的数据的难题。

- `Mark` 类定义了标记的身份（`type` + `attrs`）。
- `MarkSet`（通过 `readonly Mark[]` 和一系列规则实现）是 ProseMirror 中处理标记集合的**规范方式**。
- **排序 (`rank`)** 是实现规范表示和高效比较 (`sameSet`) 的关键。
- **不可变性**和**惰性复制** (`addToSet` 中的 `copy` 标志) 兼顾了数据模型的健壮性和性能。
- **互斥 (`excludes`)** 规则被优雅地集成在 `addToSet` 的逻辑中，使得复杂的 schema 规则得以在核心数据结构层面被强制执行。

理解了 `Mark` 和 `MarkSet` 的工作原理，你就掌握了 ProseMirror 如何处理内联格式的底层机制。
