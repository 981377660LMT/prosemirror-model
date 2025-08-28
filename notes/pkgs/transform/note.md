正在收集工作区信息好的，我们来深入讲解 prosemirror-transform 模块。

prosemirror-transform 是 ProseMirror 体系中负责处理文档变更的核心模块。它将文档的修改抽象为一系列可追踪、可逆、可重放的“步骤”（Steps），这为协同编辑、历史记录（撤销/重做）等高级功能奠定了基础。

### 1. 核心概念

- **`Transform`**: 这是该模块的中心类。一个 `Transform` 对象代表了一系列对文档的修改。它持有一个文档的初始版本和一组将要应用到该文档上的 `Step`。`Transform` 的实例是不可变的；它的每个变更方法（如 `delete`, `addMark`）都会返回一个包含新步骤的、**新的** `Transform` 实例，这使得方法可以被链式调用。

- **`Step`**: `Step` 是一个原子化的、可序列化（为 JSON）的文档变更操作。一个 `Transform` 就是由一个或多个 `Step` 组成的。最常见的步骤是 `ReplaceStep`，它描述了对文档某个范围内容的替换。

- **`Mapping`**: `Mapping` 是一个至关重要的接口，它能够描述文档在经过一次 `Transform` 之后，位置是如何变化的。例如，如果在文档开头插入了文本，那么原先在位置 10 的内容现在就在了新的位置。`Mapping` 可以告诉你这个新位置是什么。这对于在应用变更后更新光标位置、同步不同用户的操作至关重要。

### 2. `Transform` 的工作流程

一个典型的 `Transform` 工作流程如下：

1.  基于一个文档节点（`Node`）创建一个 `Transform` 实例。
2.  链式调用各种变更方法来构建一系列 `Step`。
3.  最终，`tr.doc` 属性会持有应用了所有步骤之后的新文档。`tr.steps` 包含了所有的变更步骤，`tr.mapping` 则记录了从初始文档到最终文档的位置映射关系。

下面是一个代码示例，演示了如何删除一段文本，并给另一段文本添加标记：

```typescript
// ...existing code...
import { schema, doc, p, strong } from 'prosemirror-test-builder'
import { Transform } from 'prosemirror-transform'
import { Slice } from 'prosemirror-model'

// 假设我们有初始文档
const startDoc = doc(p('one two three'))

// 1. 创建一个 Transform 实例
let tr = new Transform(startDoc)

// 2. 链式调用，构建变更
// 删除 "two " (从位置 5 到 9)
tr.deleteRange(5, 9)

// 给 "one" 添加 strong 标记 (从位置 1 到 4)
tr.addMark(1, 4, schema.marks.strong.create())

// 3. 查看结果
// tr.doc 是应用变更后的新文档
console.log(tr.doc.toJSON())
// 输出: doc(p(strong("one"), " three"))

// tr.steps 包含了两个步骤：一个 ReplaceStep(用于删除) 和一个 AddMarkStep
console.log(tr.steps.length) // 2
```

### 3. 与其他模块的关系

- **prosemirror-model**: `Transform` 的操作对象是来自 prosemirror-model 的 `Node` 和 `Slice`。它依赖 prosemirror-model 的数据结构来执行替换、分裂、合并等操作。

- **prosemirror-state**: 在实际应用中，你通常不会直接使用 `Transform`。prosemirror-state 模块中的 `Transaction` 类继承自 `Transform`，并在此基础上增加了对选区（Selection）、存储的标记（Stored Marks）和插件元数据（Metadata）的管理。在编写编辑器逻辑时，你打交道最多的是 `Transaction`。

- **prosemirror-history & prosemirror-collab**: 这两个模块是 prosemirror-transform 能力的最佳体现。`history` 模块通过反转（invert）`Step` 来实现撤销。`collab` 模块则通过交换和重映射（rebase）来自不同客户端的 `Step` 来实现协同编辑。

总而言之，prosemirror-transform 提供了一套强大而精确的文档变更描述和操作机制。它将复杂的文档修改分解为简单、可控的步骤，是 ProseMirror 实现其声明式、可预测状态管理和高级功能的基石。
