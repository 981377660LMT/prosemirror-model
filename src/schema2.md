好的，让我们以最详尽、最有逻辑的方式，从头到尾、由内而外地剖析 schema.ts 这个文件。我将扮演好我的角色，带你领略 ProseMirror 模型层的设计精髓。

这个文件是 ProseMirror 的基石。它的使命是：**将用户提供的、声明式的文档结构定义（`SchemaSpec`），编译成一个高效、严谨、可供运行时使用的规则集（`Schema` 实例）。**

我们将按照逻辑依赖关系，从最小的构建单元开始，逐步向上构建，最终理解 `Schema` 这个顶层类。

---

### 逻辑结构概览

1.  **属性（Attributes）系统**: 节点和标记都可以有属性。文件首先定义了处理属性的一套完整机制，包括定义、编译、计算和校验。
2.  **`NodeType` (节点类型)**: 定义了文档的结构元素，如段落、标题。
3.  **`MarkType` (标记类型)**: 定义了应用于内联内容的样式，如加粗、链接。
4.  **`Schema` (纲要)**: 最终的容器，它持有并管理所有的 `NodeType` 和 `MarkType`，并将它们链接在一起，形成一个完整的、自洽的规则系统。

---

### 第一部分：属性（Attributes）系统

这是最底层的构建块。一个属性可以是链接的 `href`，也可以是图片的 `src`。

#### 1. `AttributeSpec` (接口)

```typescript
// ...existing code...
export interface AttributeSpec {
  default?: any
  validate?: string | ((value: any) => void)
}
// ...existing code...
```

这是用户在定义 `NodeSpec` 或 `MarkSpec` 时提供的**原始蓝图**。它非常简单：

- `default`: 如果用户在创建节点/标记时未提供此属性，则使用该默认值。
- `validate`: 一个用于验证属性值的规则。可以是自定义函数，也可以是一个用 `|` 分隔的原始类型字符串（如 `"string|number|null"`）。

#### 2. `Attribute` (类)

```typescript
// ...existing code...
class Attribute {
  // ...
  constructor(typeName: string, attrName: string, options: AttributeSpec) {
    this.hasDefault = Object.prototype.hasOwnProperty.call(options, 'default')
    this.default = options.default
    this.validate =
      typeof options.validate == 'string'
        ? validateType(typeName, attrName, options.validate)
        : options.validate
  }
  // ...
}
// ...existing code...
```

这是 `AttributeSpec` 的**编译后形态**，是 ProseMirror 内部使用的对象。构造函数做了两件重要的事情：

1.  明确记录 `hasDefault`，因为 `default: undefined` 和没有 `default` 属性是两种不同情况。
2.  **编译 `validate`**：如果 `validate` 是字符串，它会通过 `validateType` 辅助函数生成一个真正的验证函数。这是一个重要的优化和规范化步骤，避免在每次验证时都去解析字符串。

#### 3. 属性相关的辅助函数

- `initAttrs(typeName, attrs?)`: 这是一个工厂函数。它接收一个 `AttributeSpec` 对象的映射，遍历它们，为每一个 `Spec` 创建一个 `Attribute` 类的实例，最后返回一个 `Attribute` 实例的映射。这是从“蓝图”到“内部表示”的第一步。

- `defaultAttrs(attrs)`: 这是一个**性能优化**。它遍历编译后的 `Attribute` 映射，如果所有属性都有默认值，它会创建一个包含所有默认值的**单例对象**。这样，在创建没有指定属性的节点时，可以直接复用这个对象，避免重复创建。

- `computeAttrs(attrs, value)`: 当创建一个节点或标记时，此函数负责计算最终的属性对象。它会遍历 schema 中定义的所有属性：

  - 如果用户提供了值 (`value[name]`)，则使用该值。
  - 如果用户未提供，检查是否有默认值 (`attr.hasDefault`)，有则使用。
  - 如果既未提供也无默认值（即必填属性），则抛出错误。

- `checkAttrs(attrs, values, ...)`: 此函数用于**校验**一个给定的属性对象 (`values`) 是否符合规则。它做两件事：

  1.  检查 `values` 中是否有多余的、未在 schema 中定义的属性。
  2.  对 `values` 中的每个属性值，调用其对应的 `validate` 函数进行类型和格式检查。

  ```ts
  function checkAttrs(
    attrs: { [name: string]: Attribute },
    values: Attrs,
    type: string,
    name: string
  ) {
    for (let name in values)
      if (!(name in attrs))
        throw new RangeError(`Unsupported attribute ${name} for ${type} of type ${name}`)
    for (let name in attrs) {
      let attr = attrs[name]
      if (attr.validate) attr.validate(values[name])
    }
  }
  ```

---

### 第二部分：`NodeType` - 文档的骨架

`NodeType` 代表一种节点，例如 `paragraph`。它不是节点本身，而是节点的“类型”或“类”。

#### 1. `NodeSpec` (接口)

这是用户用来定义一个节点类型的蓝图。它包含了大量配置项，例如：

- `content`: 一个用特定语法描述的**内容表达式**，如 `"inline*"` (0 个或多个内联节点) 或 `"block+"` (1 个或多个块级节点)。这是定义文档结构的核心。
- `marks`: 允许在此节点内部出现的标记类型。`"_"` 表示全部，`""` 表示没有，`"strong em"` 表示只有这两种。
- `group`: 节点所属的组，如 `"block"` 或 `"inline"`，用于在内容表达式中引用。
- `inline`: `true` 表示是内联节点（如 `image`），否则是块级节点（如 `paragraph`）。
- `atom`: `true` 表示这是一个“原子”节点，在编辑器中被视为一个不可分割的整体，即使它可能有内容（如一个复杂的可交互嵌入块）。
- `toDOM` / `parseDOM`: 定义了该节点类型如何与 DOM 进行相互转换。

#### 2. `NodeType` (类)

这是 `NodeSpec` 的编译后形态，包含了大量从 `Spec` 计算和链接而来的属性和方法。

##### `constructor(name, schema, spec)`

构造函数进行初步的属性初始化：

- 保存 `name`, `schema`, `spec` 的引用。
- 解析 `group` 字符串为数组。
- 调用 `initAttrs` 和 `defaultAttrs` 初始化属性系统。
- 根据 `spec.inline` 和 `name == 'text'` 设置 `isBlock` 和 `isText` 基础属性。
- **延迟初始化**: `contentMatch` 和 `inlineContent` 被设为 `null`。它们非常重要，但依赖于所有其他 `NodeType` 都已创建，因此会在稍后的 `Schema` **构造函数中被填充**。

##### 核心属性 (Getters & 计算属性)

- `isInline`, `isTextblock`, `isLeaf`, `isAtom`: 这些都是基于基础属性和 `contentMatch` 计算出的便捷布尔值，用于快速判断节点特性。
- `contentMatch`: 这是 `NodeType` 最强大的属性之一。它是一个 `ContentMatch` 实例（一个有限状态自动机），由 `spec.content` 字符串编译而来。它能高效地回答“这个子节点序列是否是我的合法内容？”这类问题。
- `markSet`: 由 `spec.marks` 字符串编译而来，是一个 `MarkType` 实例的数组（或 `null`），用于快速检查标记是否被允许。

##### 核心方法

- `create(attrs, content, marks)`: 创建一个此类型的**节点实例** (`Node`)。它不检查内容是否合法，因此速度更快，主要供内部使用。
- `createChecked(attrs, content, marks)`: 创建节点前会调用 `checkContent` 检查内容合法性，如果内容不符合 `contentMatch` 规则，则会抛出错误。这是更安全的、推荐外部使用的方法。
- `createAndFill(attrs, content, marks)`: 一个非常智能的方法。如果给定的 `content` 不完全符合节点的 `contentMatch`，它会尝试在 `content` 的前后**自动填充**必要的节点（例如，一个 `list_item` 必须包含一个 `paragraph`，如果你只提供了文本，它会自动用 `paragraph` 包裹文本），以使其合法。这是实现粘贴、拖拽等复杂操作的关键。
- `validContent(content)` / `checkContent(content)`: 使用 `contentMatch` 来验证一个 `Fragment` (节点序列) 是否是该类型的有效内容。
- `allowsMarkType(markType)` / `allowsMarks(marks)`: 使用 `markSet` 来验证标记是否被允许。

##### `static compile(nodes, schema)`

这个静态方法是 `Schema` 构造过程的一部分。它接收 `NodeSpec` 的 `OrderedMap`，遍历并为每个 `spec` 创建一个 `NodeType` 实例。同时，它会进行一些关键的健全性检查，例如：

- 必须存在 `topNode` (通常是 `doc`)。
- 必须存在 `text` 节点。
- `text` 节点不能有属性。

---

### 第三部分：`MarkType` - 内容的样式

`MarkType` 与 `NodeType` 非常相似，但它代表的是标记（如 `strong`, `link`）。

#### 1. `MarkSpec` (接口)

定义一个标记类型的蓝图。关键属性：

- `attrs`: 与 `NodeSpec` 中的 `attrs` 完全相同。
- `excludes`: 定义此标记与哪些其他标记互斥。例如，`code` 标记通常会排除所有其他标记 (`excludes: "_"`）。当添加一个标记时，集合中所有被它排除的标记都会被移除。
- `toDOM` / `parseDOM`: 定义与 DOM 的相互转换。

#### 2. `MarkType` (类)

`MarkSpec` 的编译后形态。

##### `constructor(name, rank, schema, spec)`

- `rank`: 一个整数，由 `MarkType.compile` 根据 `MarkSpec` 在 `marks` 映射中的顺序分配。这个 `rank` 用于在排序标记集合时提供一个稳定、确定的顺序。
- `instance`: 另一个性能优化。如果一个标记类型没有属性，或者所有属性都有默认值，那么它的实例是固定的。这里会预先创建一个**单例 `Mark` 实例**，之后通过 `create()` 方法可以直接复用它。

##### 核心方法

- `create(attrs)`: 创建一个此类型的**标记实例** (`Mark`)。如果可以，会返回缓存的 `instance`。
- `removeFromSet(set)` / `isInSet(set)`: 在一个 `Mark` 数组中移除或查找此类型的标记。
- `excludes(other)`: 检查此标记类型是否排除另一个标记类型。

##### `static compile(marks, schema)`

与 `NodeType.compile` 类似，遍历 `MarkSpec` 的 `OrderedMap`，为每个 `spec` 创建一个 `MarkType` 实例，并在这个过程中为它们分配 `rank`。

---

### 第四部分：`Schema` - 最终的集大成者

现在，我们终于来到了顶层 `Schema` 类。它的构造函数是整个文件的“主函数”，负责编排上述所有组件。

#### `constructor(spec: SchemaSpec)`

1.  **规格转换**: 将用户传入的 `spec.nodes` 和 `spec.marks`（可能是普通对象）转换为 `OrderedMap`。**顺序非常重要**，它决定了 DOM 解析的优先级和标记的 `rank`。
2.  **编译 `NodeType` 和 `MarkType`**: 调用 `NodeType.compile` 和 `MarkType.compile`，创建所有类型实例。至此，`this.nodes` 和 `this.marks` 已经填充了 `NodeType` 和 `MarkType` 对象，但它们内部的链接尚未完成。
3.  **后处理与链接（The Grand Finale Loop）**: 这是最关键的一步。构造函数会遍历所有已创建的 `NodeType` 和 `MarkType`，完成它们的“延迟初始化”和交叉链接。

    - **遍历 `this.nodes`**:
      - 检查命名冲突：确保没有一个名字同时是节点和标记。
      - **编译 `contentMatch`**: `type.contentMatch = ContentMatch.parse(...)`。它将 `content` 字符串（如 `"paragraph+"`）解析成一个状态机。因为不同节点可能共享相同的内容表达式，所以这里用 `contentExprCache` 来缓存解析结果，避免重复工作。
      - 设置 `inlineContent`: `type.contentMatch.inlineContent` 的结果被赋给 `type.inlineContent`。
      - **编译 `markSet`**: `type.markSet = gatherMarks(...)`。它将 `marks` 字符串（如 `"strong em"`）解析成一个包含真实 `MarkType` 实例的数组。`gatherMarks` 辅助函数负责处理名称和组名。
    - **遍历 `this.marks`**:
      - **编译 `excluded`**: `type.excluded = gatherMarks(...)`。与 `markSet` 类似，它将 `excludes` 字符串解析成一个 `MarkType` 数组。

4.  **收尾工作**:
    - 设置 `topNodeType`。
    - 创建 `nodeFromJSON` 和 `markFromJSON` 这两个绑定了当前 schema 实例的便捷方法。
    - 初始化 `cached` 对象，用于给其他模块（如 prosemirror-transform）缓存与此 schema 相关的数据。

### 总结

schema.ts 的设计展现了一个清晰的“编译”思想：

1.  **用户提供声明式的、易于理解的蓝图 (`Spec` 对象)。**
2.  **ProseMirror 将这些蓝图编译成内部使用的、高度优化和链接的运行时对象 (`NodeType`, `MarkType`)。**
3.  **最终，`Schema` 类作为总指挥，持有所有这些编译后的规则，并提供一套 API 来创建和验证符合这些规则的文档内容。**

这种将“定义”与“执行”分离的架构，使得 ProseMirror 的核心既灵活（易于通过 Spec 定制）又高效（运行时检查速度快），并保证了数据模型的绝对严谨性。
