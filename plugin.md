# 插件体系

Slate.js 通过插件体系扩展富文本编辑器：

```js
const BoldPlugin = () => ({
  // register commands
  commands: {
    toggleBold: () => {}
  },
  // register queries
  queries: {
    isBoldActive: () => {}
  },
  // normalizer schema
  schema: {},
  // event handlers
  onKeyDown,
  onMouseDown,
  // renderers
  renderMark,
  renderNode
})

const ListPlugin = () => ({
  // ...
})

const plugins = [
  BoldPlugin(),
  ListPlugin(),
  // ...
]
```

插件支持的配置可以分为两类：

- **能力注入**：commands，queries，schema
- **监听钩子**：onKeyDown 这类的事件 UI Event Hook，以及 renderNode 这类渲染 Hook

因此，Slate.js 在注册一个插件到编辑器实例上时，也是分为两个阶段：

- 阶段一：将插件注入的能力绑定到编辑器 Controller 实例对象
- 阶段二：将插件声明的 Hook 添加到对应的 middleware 末尾

<p align="center">
  <img src="./statics/plugin-structure.png" />
</p>

```js
// packages/slate/src/controllers/editor.js

function registerPlugin(editor, plugin) {
  if (Array.isArray(plugin)) {
    plugin.forEach(p => registerPlugin(editor, p))
    return
  }

  const { commands, queries, schema, ...rest } = plugin

  if (commands) {
    const commandsPlugin = CommandsPlugin(commands)
    registerPlugin(editor, commandsPlugin)
  }

  if (queries) {
    const queriesPlugin = QueriesPlugin(queries)
    registerPlugin(editor, queriesPlugin)
  }

  if (schema) {
    const schemaPlugin = SchemaPlugin(schema)
    registerPlugin(editor, schemaPlugin)
  }

  for (const key in rest) {
    const fn = rest[key]
    const middleware = (editor.middleware[key] = editor.middleware[key] || [])
    middleware.push(fn)
  }
}
```

## Middleware

Slate.js 的插件体系参考了 [koa.js]()，将编辑器中可以被扩展的位置提取为了一个个的 Hook，每一个 Hook 都对应了中间件，它们被绑定到了编辑器 Controller 实例上的 `middleware` 属性上：

```js
editor.middleware = {
  onKeyDown: [boldOnKeyDown, listOnKeyDown],
  renderNode: [boldRenderNode, listRenderNode],
  // ...
}
```

并通过 `editor.run(hookName, ..args)` 来执行对应的 middleware：

```js
// packages/slate/src/interfaces/node.js
class NodeInterface {
  // ...
  normalize(editor) {
    const normalizer = editor.run('normalizeNode', this)
    return normalizer
  }
}
```

`editor.run` 的过程和 koa.js 的 middleware 类似：使用迭代器的思路依序执行 middleware 序列中的各个 hook。Slate.js 的每个 hook 也会被注入当前编辑器 Controller 实例和 next 方法，用于控制是否继续 middleware 中的下一个过程：

```js
// packages/slate/src/controllers/editor.js
class Editor {
  // ...
  run(key, ...args) {
    const { controller, middleware } = this
    const fns = middleware[key] || []
    let i = 0

    function next(...overrides) {
      const fn = fns[i++]
      if (!fn) return

      if (overrides.length) {
        args = overrides
      }

      const ret = fn(...args, controller, next)
      return ret
    }


    return next()
  }
}
```

在插件声明的 hook 中，我们可以通过 `next()` 继续 middleware 的执行，也可以直接返回 `controller` 阻断当前 middleware 的执行：

```js
const BoldPlugin = () => {
  onKeyDown: (event, controller, next) => {
    if (isBoldHotKey(event)) {
      return controller.toggleMark('bold')
    }
    return next();
  }
}
```

## 优先级与冲突

Slate.js 的插件组织，每个插件的优先级取决如插件在 `plugins` 队列中的位置，最早被放入的插件，其下声明的所有 hook 都能被最早处理。

一旦我们的插件数目变多，就会面临怎么组织插件的问题：对于插件 A、插件 B 和插件 C，能够正确工作的 `onKeyDown` 顺序是 ABC，能够正确工作的 `onMouseDown`又是 BAC。因此对插件的仅仅做单元测试不够的，还应当「对应用程序所使用的插件集」做集成测试。

Slate.js 在 0.50 以后，放弃了使用 middleware 形式的插件体系：hook 的声明「内联在了一个位置」：

```jsx
const App = () => {
  const editor = useMemo(() => withReact(createEditor()), [])
  const [value, setValue] = useState([
    {
      type: 'paragraph',
      children: [{ text: 'A line of text in a paragraph.' }],
    },
  ])

  return (
    <Slate editor={editor} value={value} onChange={value => setValue(value)}>
      <Editable
        onKeyDown={event => {
          if (event.key === '&') {
            // Prevent the ampersand character from being inserted.
            event.preventDefault()
            // Execute the `insertText` method when the event occurs.
            editor.insertText("and")
          }
        }}
      />
    </Slate>
  )
}
```

但是这也不是完美的解法，不只是代码中肉眼所见的臃肿的 Event Handler 定义，更严重的是，这种松散的结构「不利于编辑器能力的插拔」，有朝一日我们想下线某个能力时，是无法整体下线的，需要在各个 Handler 中删除逻辑。