# Controller

Controller 是 Slate.js 大脑，在新版本里，它被重命名为 Editor，无论是何种名字，他反映的都是一个编辑器实例对象。

<p align="center">
  <img src="./statics/controller.png" width="500" />
</p>

开发者可以通过实例化一个 Controller 对象，实现对文档内容的访问和控制：

```js
import { Editor } from 'slate'

const controller = new Editor({
  value,
  plugins,
  readOnly,
  onChange
}, { normalize: false });
```

可以看到，初始化一个编辑器，需要告诉 Slate.js:

- `value`：编辑器的初始值
- `plugins`：编辑器具有哪些能力，即为编辑器注册哪些插件，我们会在 [插件](./plugin.md) 一节中对其做更详尽的描述
- `readOnly`：编辑器是否为只读模式
- `onChange`：当文档模型变更后，应当作何反应。例如，我们可以刷新编辑器内容，也可以向协同者广播变更产生的操作集合

另外也支持对初始化过程做一些配置，例如：

- `normalize`：是否对文档进行 normalize，这个概念我们将在 [normalize](./normalize.md) 一节中阐述

Controller 是 UI 无关的，它的实例方法都是用来操作文档模型（内容及选区）的：

```js
controller
  .insertText('Hello World')
  .moveToStartOfNextText()
```

这些方法在 Slate.js 被命名为 Command，即指令，接下来我们就看看 Slate.js 是如何设计指令系统的。



