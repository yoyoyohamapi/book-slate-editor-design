# Annotation

Decoration 虽然为节点提供了动态装饰的能力，但是它却有一些限制：

- **歧义**：以 Mark + renderMark 作为装饰手段，开发者难于理清它与文档模型中的 Mark 的异同
- **不支持协同**：假定我们通过 `setDecorations(decorations)`  为一个图片节点装饰上了 “上传进度”，期望协同者也能看到这个进度，基于 Decoration 实现起来就比较困难了，因为 decorations 无法通过 Operation 记性协同

Slate.js 在 0.47 版本后引入了 Annotation 来替代 Decoration，它的数据结构被描述为：

```ts
{
  object: 'annotation',
  key: String,
  type: String,
  data: Map,
  anchor: Point,
  focus: Point,
}
```

<p align="center">
  <img src="./statics/annotation.png" width="700" />
</p>

为了增加灵活性，减小语义干扰，Annotation 不再使用 Mark 作为装饰类型，而是用了 `type` 来定义装饰类型，也能通过 `data` 为节点注入装饰数据。

Slate.js 也提供了 `renderAnnotation` hook 让插件自定义 Annotation 的渲染方式：

```jsx
renderAnnotation = (props, editor, next) => {
  const { children, annotation, attributes } = props

  switch (annotation.type) {
    case 'highlight':
      return (
        <span {...attributes} style={{ backgroundColor: '#ffeeba' }}>
          {children}
        </span>
      )
    default:
      return next()
  }
}
```

通过调用编辑器实例的 `addAnnotation(annotation)` 方法能够新增一条 Annotation：

```js
editor.addAnnotation({
  key: getHighlightKey(),
  type: 'highlight',
  anchor: { path, key, offset: offset - string.length },
  focus: { path, key, offset },
})
```

## 内部实现

Annotation 工作流程与 Decoration 类似，为了避免遍历全量 annotations 的开销，在渲染节点期间，会为节点孩子分发属于它自己的 annotations。与 Decoration 不同的是，节点拥有的 annotations 都来自于注入，自身无法通过类似 `decorateNode` 这样的 hook 生成。

更具体地，Slate.js 注入并渲染 annotations 的过程为：

1. 首先为根节点注入 top-level 的 annotations（`controller.value.annotations`）
2. 渲染 Node 期间，会分发属于其自己的 decorations 到孩子节点
3. 直到 annotations 被分发到了文本节点，会根据 annotations 拆分出不同的 Leaf
4. 渲染 Leaf 期间，插件的 `renderAnnotation` 被调用

<p align="center">
  <img src="./statics/annotations-arrangement.png" width="500" />
</p>

## 支持协同

更进一步，Slate.js 为 Annotation 附带了增删改查的 Operation，如 `add_annotation` 和 `remove_annotation`，使得 Annotation 支持协同。假定我们的编辑器需要展示协同者的选区，那么就可以将协同选区定义为一个 Annotation：

```js
{
  object: 'annotation',
  key: 'user id',
  type: 'remote_selection',
  data: {
    userid: 'user id',
    avatar: 'user-avatar-url'
  },
  anchor,
  focus
}
```

每当协作者的选区变更时，`renderAnnotation` 都会被调用，协作者的选区被绘制到编辑器上。想象力远不止于此，基于 Annotation，我们还能实现多人实时划词评论等等。

