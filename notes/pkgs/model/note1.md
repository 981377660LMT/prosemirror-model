好的，我们直接深入代码实现的核心，一针见血地分析 prosemirror-model 的关键机制。

### 1. DOM 解析 (`from_dom.ts`): 从 HTML 到文档节点的精密转换

DOM 解析的核心是 `DOMParser` 和 `ParseContext` 两个类，它们协同工作，将非结构化的 DOM 树转换为符合 Schema 约束的 ProseMirror 节点树。

#### **`DOMParser`：规则的调度中心**

1.  **规则编译**: `DOMParser.fromSchema` (`from_dom.ts:314`) 是入口。它调用 `schemaRules` (`from_dom.ts:284`) 遍历 Schema 中所有 NodeSpec 和 MarkSpec 的 `parseDOM` 属性，收集所有解析规则。
2.  **规则排序与分类**: 收集到的规则按 `priority`（默认为 50）降序排列。在构造函数 (`from_dom.ts:200`) 中，规则被分为 `tags` (按标签名匹配) 和 `styles` (按 CSS 样式匹配) 两个数组，便于后续快速查找。
3.  **解析启动**: `parse` (`from_dom.ts:219`) 或 `parseSlice` (`from_dom.ts:226`) 方法创建一个 `ParseContext` 实例，这是整个解析过程的状态机。

#### **`ParseContext`：状态化的树构建器**

`ParseContext` (`from_dom.ts:457`) 维护一个 `nodes` 栈，栈中的每个元素 (`NodeContext`) 代表一个正在构建的 ProseMirror 节点。

1.  **DOM 遍历**: `addAll` (`from_dom.ts:682`) 递归遍历 DOM 节点。文本节点由 `addTextNode` (`from_dom.ts:511`) 处理，元素节点由 `addElement` (`from_dom.ts:548`) 处理。
2.  **规则匹配**:
    - `addElement` 调用 `parser.matchTag` (`from_dom.ts:238`) 来查找匹配当前 DOM 元素的最高优先级的 `tag` 规则。匹配时会检查 `context` 限制。
    - `readStyles` (`from_dom.ts:611`) 遍历规则中声明的样式，调用 `parser.matchStyle` (`from_dom.ts:259`) 应用 `style` 规则，通常用于添加 Mark。
3.  **内容填充与结构调整**:
    - 当一个节点需要被插入时，`findPlace` (`from_dom.ts:703`) 被调用。它会查询当前 `NodeContext` 栈顶节点的 `ContentMatch`，看是否能直接接受新节点。
    - 如果不能，`findWrapping` (`from_dom.ts:408`) 会被调用，尝试找到一个能够包裹新节点的、符合 Schema 的节点类型序列（例如，将裸文本包裹在 `<p>` 中）。
    - 找到路径后，通过 `enterInner` (`from_dom.ts:757`) 创建并推入新的 `NodeContext` 到栈中，完成结构包裹。
4.  **空白处理**: `wsOptionsFor` (`from_dom.ts:376`) 和 `addTextNode` (`from_dom.ts:511`) 中的逻辑精确控制空白字符的保留或折叠，基于节点的 `whitespace` 属性和解析选项。
5.  **完成**: `finish` (`from_dom.ts:791`) 方法从内到外关闭所有打开的 `NodeContext`，最终返回一个完整的 `Node` 或 `Fragment`。

### 2. DOM 序列化 (`to_dom.ts`): 从节点到 HTML 的结构化输出

`DOMSerializer` (`to_dom.ts:31`) 将 ProseMirror 节点转换回 DOM。

1.  **`toDOM` 规范**: 核心是 NodeSpec 和 MarkSpec 中的 `toDOM` 函数。它返回一个 `DOMOutputSpec` (`to_dom.ts:19`)，这是一个描述 DOM 结构的数组，例如 `["p", 0]`。
2.  **渲染引擎 `renderSpec`**: 这个静态方法 (`to_dom.ts:223`) 是 `DOMOutputSpec` 的解释器。它创建标签、设置属性，并处理内容“洞”（`0`）。这个“洞”是关键，它标记了子节点或 Mark 的内容应该被插入的位置。
3.  **序列化过程**:
    - `serializeFragment` (`to_dom.ts:54`) 遍历 Fragment 中的每个节点。
    - 对于每个节点，它首先调用 `serializeNode` (`to_dom.ts:104`)。
    - `serializeNode` 先调用 `serializeNodeInner` (`to_dom.ts:89`) 来渲染节点本身（不含 Mark）。`serializeNodeInner` 内部调用 `renderSpec` 创建节点 DOM。如果 `toDOM` 规范返回了 `contentDOM`，则子节点会被递归序列化到这个 `contentDOM` 中。
    - 然后，`serializeNode` 从内到外遍历节点上的 `marks`，调用 `serializeMark` (`to_dom.ts:121`)，用 `renderSpec` 创建 Mark 对应的 DOM 结构，并将已生成的节点 DOM 包裹进去。
    - `serializeFragment` 还会智能地处理相邻节点的 Mark，合并相同的 Mark 标签，避免生成如 `<em>a</em><em>b</em>` 这样的冗余结构。

### 3. 内容替换 (`replace.ts`): 不可变数据结构的核心操作

所有文档修改最终都归结为 `replace` 操作。

1.  **`Slice` 对象**: 替换操作的核心数据结构是 `Slice` (`replace.ts:63`)。它不仅包含一个 `Fragment` (要插入的内容)，还包含 `openStart` 和 `openEnd` 两个深度值。这两个值表示切片两侧的节点被“切开”的深度，这使得跨多层级粘贴（如将列表项粘贴到段落中）成为可能。
2.  **替换算法**: `replace` 函数 (`node.ts:191`) 是一个复杂的递归过程。
    - 它从文档树的顶层开始，向下递归，直到找到包含 `from` 和 `to` 范围的节点。
    - 在目标层级，它尝试将 `from` 左侧的内容、`Slice` 的内容、`to` 右侧的内容拼接起来。
    - **关键在于拼接**：算法会利用 `openStart` 和 `openEnd` 深度，尝试合并相邻的、类型兼容的节点。例如，`p("a")` 和 `p("b")` 在拼接时会合并成 `p("ab")`。这个过程会检查 Schema 的 `contentMatch`，确保合并后的父节点内容仍然合法。
    - 如果内容不兼容，它会向上回溯，尝试在更上层的节点中进行拼接。这个过程确保了替换总是在最深的可能层级上进行，以保持文档结构的稳定性。

### 4. 内容约束 (`content.ts`): Schema 的守护者

`ContentMatch` (`content.ts`) 是一个基于 Schema 内容表达式（如 `"paragraph+ (blockquote | heading)?"`）编译出的有限状态机。

1.  **编译**: `ContentMatch.parse` (`content.ts:229`) 将字符串表达式解析成一个 `ContentMatch` 实例网络，每个实例代表一个状态。
2.  **验证**: `matchType(type)` 方法消费一个节点类型，如果匹配成功，则返回下一个状态（另一个 `ContentMatch` 实例）；否则返回 `null`。`validEnd` 属性表示当前状态是否是合法的终止状态。
3.  **修复**:
    - `fillBefore(after, toEnd)` 是最强大的功能之一。它计算出一个 `Fragment`，用于填充在当前状态和 `after` 片段之间，使得拼接后的内容合法。这在节点分割或粘贴时用于自动插入必需的节点（如在两个 `list_item` 之间自动补上 `paragraph`）。
    - `findWrapping(type)` 用于寻找能够包裹给定 `type` 的节点类型数组，使得它能被插入到当前位置。这是实现自动包裹（如输入时将文本放入段落）的基础。

这些模块紧密协作，构成了一个健壮、可预测且高性能的富文本数据模型。其核心是**不可变数据结构**、**基于状态机的 Schema 验证**和**精密的递归替换算法**。
