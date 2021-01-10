# 指令系统

在 [序言](./[preface..md]) 一节中，我们就讨论过，`document.execCommand` 在各个浏览器上实现各异，因此，流行的富文本编辑器都不会依赖于原生的 `document.execCommand` 操作编辑器，而是会「自行划归指令集范围」。

<p align="center">
  <img src="./statics/no-document-execcommand.png" width="500" />
</p>

Slate.js 也是如此，当我们拥有了一个编辑器实例，就能通过绑定在其上的各种指令（Command）来操控编辑器，不仅能修改文档的内容，也能修改文档的选区：

```js
controller
  .insertText('Hello World.') // 修改内容
  .moveToStartOfNextText();   // 移动选区
```

Slate.js 也为指令做了不同类型的划分：

- 指定区间的指令：如 `removeTextAtRange()`，`deleteAtRange()` 等
- 作用于当前选区的指令：如 `delete()`，`addMark()` 等
- 设置选区的指令：如 `focus()`，`moveToStartOfDocument()` 等
- 作用于特定节点的指令：如 `insertNodeByPath()`，`removeNodeByKey()` 等
- 作用于整个文档的指令：如 `setData()`，`setDecoration()` 等
- 历史记录相关的操作：如 `undo()`，`redo()` 等

## 注册指令

除了内置的指令，Slate.js 也支持通过插件扩展指令，插件添加的指令会在「运行时」注入到 Controller 实例上：

```js
const BoldPlugin = () => {
  commands: {
    toggleBold: (controller) => {
      controller.toggleMark('bold')
    }
  }
}

const controller = new Controller({
  value,
  plugins: [BoldPlugin()]
})

// 现在，可以使用 `toggleBold` 命令来加粗或者取消加粗文本
controller.toggleBold()
```

## 绑定指令

出于减少冗余代码的考虑，Slate.js 大量的命令都是在运行时动态注入的，例如 `*ByKey`  指令就是基于 `*ByPath` 指令创建，再注入到 Controller 实例的：

```js
/**
 * Mix in `*ByKey` variants.
 */

const COMMANDS = [
  'addMark',
  'insertFragment',
  'insertNode',
  'insertText',
  'mergeNode',
  'removeAllMarks',
  'removeMark',
  'removeNode',
  // ...
]

for (const method of COMMANDS) {
  Commands[`${method}ByKey`] = (editor, key, ...args) => {
    const { value } = editor
    const { document } = value
    const path = document.assertPath(key)
    editor[`${method}ByPath`](path, ...args)
  }
}
```

这种动态绑定方式，虽然砍掉了不少样板代码，但是对于代码提示并不友好，IDE 或者代码编辑器无法提示某个指令是否存在（这当然不绝对，因为社区可以为 TypeScript 用户提供一份 `.d.ts`）。更为严重的是，这不是一个安全的设计，例如如果我们没有注册加粗插件，那么调用 `controller.toggleBold` ，就会引发错误：

```txt
Uncaught TypeError: controller.toggleBold is not a function
```

## Query

另一类特殊的指令是 Query 指令，也就是查询指令，用于查询当前文档信息：

```js
const isVoid = controller.isVoid(node)
```

同样地，也支持通过插件扩展 query，扩展的指令同样在运行时绑定到编辑器实例上：

```js
const BoldPlugin = () => {
  queries: {
    isBoldActive: (controller) => {}
  },
}

controller.isBoldActive();
```

## Operation

在 [Slate.js 工作过程]() 一节中，我们看到，调用指令后，生成了一系列的操作（Operation），这些操作最终逐个应用在文档模型上，才生成了新的文档模型。那么，Slate.js 为什么不直接在调用指令的时候就修改文档内容呢？而是还需要 Operation 这个中间产物呢，下一节中，我们将给出 Slate.js 这么设计的理由。