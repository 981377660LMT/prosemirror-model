prosemirror-model 是 ProseMirror 的核心模块之一，它定义了用于表示和操作文档的数据结构。可以把它理解为编辑器文档内容的“骨架”和“语法规则”。

下面是 prosemirror-model 的几个核心概念：

### 1. 文档结构 (Document Structure)

ProseMirror 的文档是一个树形结构。这些数据结构都是**不可变 (immutable)** 的，任何修改都会返回一个新的对象，而不是在原地修改，这使得状态管理和协同编辑变得更加容易。

- **`Node`**: 文档的基本构建块。整个文档本身是一个顶层 `Node`，它的子节点也是 `Node`。每个节点都有一个类型（如 `paragraph`、`heading`）和一些属性。例如，`image` 节点可能有一个 `src` 属性。

- **`Fragment`**: 代表一系列相邻的 `Node`。一个非叶子节点的 `content` 就是一个 `Fragment`。

- **`Mark`**: 附加到内联内容（如文本）上的“标记”，用于添加额外样式或信息，例如加粗 (`strong`)、斜体 (`em`) 或链接 (`link`)。

- **`Slice`**: 表示文档的一个“切片”，即文档的一部分。它通常用于复制、剪切和粘贴操作。`Slice` 的两端可以是“开放”的，这在粘贴列表项到另一个列表中时非常有用。

### 2. Schema (文档模式)

Schema 定义了特定文档中允许存在哪些类型的节点和标记，以及它们之间的关系（例如，哪些节点可以作为哪些节点的子节点）。它就像文档的“语法”。

- **`Schema`**: 整个模式的容器。通过一个 `SchemaSpec` 对象来构造，该对象定义了所有的 `nodes` 和 `marks`。

- **`NodeSpec` & `MarkSpec`**: 用于定义单个节点类型和标记类型的配置对象。在这里你可以定义节点的 `content` 表达式、属性 (`attrs`)、如何解析和序列化到 DOM 等。

- **`NodeType` & `MarkType`**: `NodeSpec` 和 `MarkSpec` 在 `Schema` 实例化后的运行时表示。

- **`ContentMatch`**: 一个非常强大的内部机制，由节点的 `content` 表达式（例如 `"paragraph+"` 或 `"block*"`）编译而来。它用于在编辑过程中高效地检查一个节点的内容是否仍然符合其类型在 Schema 中定义的规则。

### 3. DOM 交互 (DOM Representation)

prosemirror-model 提供了将文档模型与浏览器 DOM 进行相互转换的工具。

- **`DOMParser`**: 负责将 DOM 树解析成 ProseMirror 的 `Node` 对象。它的行为由 Schema 中每个 `NodeSpec` 和 `MarkSpec` 的 `parseDOM` 规则驱动。这些规则定义了如何将特定的 DOM 元素或样式映射回 ProseMirror 的节点或标记。

- **`DOMSerializer`**: 负责将 ProseMirror 的 `Node` 对象序列化为 DOM 树。它的行为由 `NodeSpec` 和 `MarkSpec` 的 `toDOM` 规则驱动，这些规则定义了如何为每个节点或标记创建对应的 DOM 结构。

- **`ParseContext`**: 在 `DOMParser` 工作时内部使用的一个辅助类，它维护一个节点栈来表示正在构建的 ProseMirror 节点树的当前状态。

总而言之，prosemirror-model 为 ProseMirror 编辑器提供了一个健壮、结构化的文档模型。它通过 Schema 保证了文档的合法性，并通过 Parser 和 Serializer 实现了模型与浏览器 DOM 之间的解耦和双向转换，这是 ProseMirror 强大功能和灵活性的基础。

---

正在收集工作区信息当然，这里有一个 prosemirror-model 的代码示例，它演示了如何定义一个 Schema、解析 HTML、然后将其序列化回 DOM。

这个例子将：

1.  定义一个包含 `doc`、`paragraph`、`text` 节点和 `strong` (加粗) 标记的简单 Schema。
2.  使用 `DOMParser` 将一个 HTML 字符串 (`<p>Hello, <strong>world</strong>!</p>`) 解析成一个 ProseMirror 文档节点。
3.  使用 `DOMSerializer` 将该文档节点序列化回一个 DOM 元素。

```typescript
import { Schema, DOMParser, DOMSerializer, Node as ProseMirrorNode } from 'prosemirror-model'

// 1. 定义一个 Schema，规定文档中允许出现的节点和标记
const mySchema = new Schema({
  nodes: {
    doc: {
      content: 'paragraph+'
    },
    paragraph: {
      content: 'text*',
      toDOM() {
        return ['p', 0]
      },
      parseDOM: [{ tag: 'p' }]
    },
    text: {}
  },
  marks: {
    strong: {
      toDOM() {
        return ['strong', 0]
      },
      parseDOM: [{ tag: 'strong' }, { style: 'font-weight=bold' }]
    }
  }
})

// 2. 使用 DOMParser 将 HTML 解析为 ProseMirror 节点
// 创建一个临时的 DOM 元素来容纳我们的 HTML 字符串
const domNode = document.createElement('div')
domNode.innerHTML = '<p>Hello, <strong>world</strong>!</p>'

const parser = DOMParser.fromSchema(mySchema)
const docNode: ProseMirrorNode = parser.parse(domNode)

// 打印出解析后的节点 JSON 结构
console.log(docNode.toJSON())
// 输出:
// {
//   "type": "doc",
//   "content": [
//     {
//       "type": "paragraph",
//       "content": [
//         { "type": "text", "text": "Hello, " },
//         { "type": "text", "marks": [{ "type": "strong" }], "text": "world" },
//         { "type": "text", "text": "!" }
//       ]
//     }
//   ]
// }

// 3. 使用 DOMSerializer 将 ProseMirror 节点序列化回 DOM 元素
const serializer = DOMSerializer.fromSchema(mySchema)
const renderedDOM = serializer.serializeFragment(docNode.content)

// 将渲染后的 DOM 添加到页面中以供查看
document.body.appendChild(renderedDOM)

// document.body 中现在会包含: <p>Hello, <strong>world</strong>!</p>
console.log(document.body.innerHTML)
```

这个例子展示了 prosemirror-model 的核心流程：

- **Schema** 是蓝图，定义了文档的结构和规则。
- **`DOMParser`** 是输入管道，它根据 Schema 的 `parseDOM` 规则将外部内容（如 HTML）转换为结构化的 ProseMirror 文档。
- **`DOMSerializer`** 是输出管道，它根据 Schema 的 `toDOM` 规则将内部的 ProseMirror 文档转换为可供渲染的格式（如 DOM）。
