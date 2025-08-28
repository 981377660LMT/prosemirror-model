正在收集工作区信息好的，我们来深入讲解 prosemirror-history 模块。

prosemirror-history 为 ProseMirror 编辑器提供撤销（undo）和重做（redo）功能。它的实现非常精妙，因为其历史记录是**选择性**的，而不仅仅是回滚到之前的某个完整状态。这种设计对于支持协同编辑至关重要，因为需要撤销自己的操作，同时保留其他协作者在此期间提交的更改。

### 1. 核心概念

- **`history`**: 这是模块的主要导出，一个 `Plugin` 工厂函数。你需要将这个插件添加到编辑器的 state 中来启用历史记录功能。它可以接受 `depth`（历史深度）和 `newGroupDelay`（新事件组的延迟时间）作为配置。

- **`HistoryState`**: 这是历史插件所管理的状态对象。它主要包含两个部分：`done`（一个用于撤销的 `Branch`）和 `undone`（一个用于重做的 `Branch`）。

- **`Branch`**: 代表一个历史分支（撤销栈或重做栈）。它不是一个简单的步骤数组，而是一个 `RopeSequence`（一种高效的序列数据结构），其中包含了一系列的 `Item`。

- **`Item`**: 这是历史记录中的原子单元。一个 `Item` 可以包含：
  - 一个可逆的 `Step`（实际的文档变更）。
  - 一个 `StepMap`（用于在应用历史步骤时，将其变换（map）到当前文档的正确位置）。
  - 一个 `SelectionBookmark`（标记一个“事件”的开始，即一组应被一起撤销/重做的变更）。

### 2. 工作原理

1.  **事件分组**: ProseMirror 会将连续的、短时间内的修改（由 `newGroupDelay` 控制）组合成一个“撤销事件”。例如，连续输入文本会被视为一个事件，一次撤销就会全部删除。你也可以通过 `closeHistory` 命令强制开启一个新的事件组。

2.  **记录变更**: 当一个事务（`Transaction`）被应用时，如果它包含可被记录的步骤，历史插件的 `applyTransaction` 逻辑会启动。它会计算出该事务的**反向步骤**（inverted steps），并将这些步骤连同映射信息打包成 `Item`，存入 `done` 栈（`Branch`）中。

3.  **执行撤销 (`undo`)**:

    - 当 `undo` 命令被触发时，插件会从 `done` 栈中取出最近的一个事件。
    - 它将这个事件中的步骤应用到一个新的 `Transaction` 中。**关键在于**：这些步骤在应用前，会通过存储的 `StepMap` 进行变换，以适应在原始操作之后发生的、但不应被撤销的其他变更（例如来自协作者的变更）。
    - 执行撤销后，被撤销的事件（及其反向操作）会被移入 `undone` 栈，以备重做。

4.  **协同编辑 (`collab`) 集成**:
    - 当协同编辑模块接收到远程变更并将其 rebase 到本地时，它会通知历史插件。
    - 历史插件会调用其 `Branch` 上的 `rebased` 方法。这个方法不会丢弃历史记录，而是将远程变更的 `Mapping` 添加到历史 `Item` 中。
    - 这样，即使用户的文档因为协同操作发生了很大变化，历史记录中的步骤在未来执行撤销时，依然能通过这些累积的映射被正确地应用到文档的最新位置上。

### 3. 主要 API

- **`history(config?: HistoryOptions)`**: 创建历史插件实例。
- **`undo` / `redo`**: 用于撤销和重做的命令。
- **[`undoDepth(state: EditorState)`](prosemirror-history/src/history.ts)**: 返回当前可撤销的事件数量。
- **[`redoDepth(state: EditorState)`](prosemirror-history/src/history.ts)**: 返回当前可重做的事件数量。

总而言之，prosemirror-history 通过将变更分解为带映射信息的 `Item`，并利用 prosemirror-transform 的映射（Mapping）和变换（Transform）能力，实现了一个强大且支持协同编辑的选择性历史记录系统。
