# 插件体系

在 Slate.js 中，插件是一等公民（First-Class），编辑器的任何能力都是通过插件实现的：

```js
// 加粗插件
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

// 列表插件
const ListPlugin = () => ({
  // ...
})

const plugins = [
  BoldPlugin(),
  ListPlugin(),
  // ...
]
```

插件支持的配置可以分为：

- **编辑器控制**：
  - Commands: 自定义指令
  - Queries: 自定义查询
  - Schema: 自定义节点模式
- **事件劫持**：
  - onXXXEvent: 可以劫持用户产生的 keydown, mousedown 等事件
- **节点渲染**：
  - renderNode: 自定义节点渲染
  - renderMark：自定义文本格式渲染
  - renderContent：自定义编辑器渲染

## 插件的内部组织：Middleware

因此，Slate.js 为编辑器实例注入插件分为两个阶段：

- 阶段一：将插件注入的 commands, queries, schema 绑定到编辑器 Controller 实例对象
- 阶段二：为不同的 hooks 创建不同的 middleware，并将插件声明的 hook 放入 middleware 末尾

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

Slate.js 的插件体系参考了 [koa.js]()，将编辑器中可以被扩展的位置提取为了一个个的 hook，每一个 hook 都对应了中间件，它们被绑定到了编辑器 Controller 实例上的 `middleware` 属性上：

```js
editor.middleware = {
  onKeyDown: [boldOnKeyDown, listOnKeyDown],
  renderNode: [boldRenderNode, listRenderNode],
  // ...
}
```

当某个事件到来时，通过 `editor.run(hookName, ...args)` 来执行对应的 middleware：

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

middleware 的执行过程和 koa.js 类似：使用迭代器「依序执行」 middleware 序列中的各个 hook。同样地，每个 hook 也会被注入一个 `next` 函数，用于控制是要继续，还是阻断当前事件被下一个插件处理：

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

在插件声明的 hook 中，我们可以通过 `next()` 继续 middleware 的执行，也可以直接返回 `controller` 阻断事件传播到下一个插件：

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

Slate.js 的插件组织，每个插件的优先级取决于「插件在 `plugins` 队列中的位置」，最早被放入的插件，它声明的「所有 hook」 都能被最早处理：

```js
// bold plugin 的优先级最高
const plugins = [
  boldPlugin,
  listPlugin,
  imagePlugin,
]
```

一旦插件数目变多，组织这些插件就会非常头疼，假设插件 A、B、C 都拦截了 `onKeyDown` 和 `onMouswDown`，但能让 `onKeyDown` 正确工作的顺序是 [ABC]，但要让 `onMouseDown` 正确工作的顺序确实 [CBA]。这同样影响到了测试：对插件做单元测试是不够的，还需要对「插件序列」做集成测试。

Slate.js 在 0.50 以后，放弃了 middleware 形的插件体系，为编辑器扩展 commands，queries 使用组合（Composition）模型实现：

```ts
import { createEditor } from 'slate';

const withImages = editor => {
  const { isVoid } = editor;

  editor.isVoid = element => {
    return element.type === 'image' ? true : isVoid(element);
  }

  return editor;
}

const editor = withImages(createEditor());
```

Event Hook 和 Renderer Hook 则内联到编辑器组件进行生命，不再以插件进行拆分：

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

这种风格虽然带来了更加自由的组织方式，但却「不利于编辑器能力的插拔」：当我们想扩展或者移除某个能力时，只能修改对应的 Event Handler。对于大型编辑器来说，这就会成为能力扩展的掣肘。因此，社区也提出了自己的解决方案，例如 [slate-plugins](https://github.com/udecode/slate-plugins) 约定了插件仍然将 `onKeyDown` 等 Hook 实现在插件内部：

```typescript
/**
 * - Shift+Tab: outdent code line.
 * - Tab: indent code line.
 */
export const getCodeBlockOnKeyDown = (): OnKeyDown => (editor) => (e) => {
  if (e.key === 'Tab') {
    const shiftTab = e.shiftKey;
    // 通过 return false 实现流程阻断
    return false;
  }
};
```

在 slate-plugins 内部，则通过组合的方式，串联这些插件的 Hook：

```js
/**
 * @see {@link OnKeyDown}
 */
export const pipeOnKeyDown = (
  editor: SPEditor,
  plugins: SlatePlugin[] = []
): EditableProps['onKeyDown'] => {
  const onKeyDowns = plugins.flatMap(
    (plugin) => plugin.onKeyDown?.(editor) ?? []
  );

  return (event) => {
    onKeyDowns.some((onKeyDown) => onKeyDown(event) === false);
  };
};
```

但这样的插件范式和调度方式，也又让插件回到了 Slate.js 0.4x，因此，时至今日，如何更好地组织编辑器插件，仍然困扰着 Slate.js 的开发者。