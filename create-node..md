# 标识节点

我们可以通过 `Node.create(attrs)` 来创建节点，例如要创建一个 Inline 节点，就可以：

```js
const inline = Inline.create({
  type: 'image',
  data: {
    width: 300,
    height: 300,
    src: 'https://image-source'
  }
})
```

## key

当节点被创建后，它就会被一个名为 `key` 的属性所标识，这个属性我们可以在创建的时候自行制定，若不指定，Slate 会通过内部的 `KeyUtils.create()` 生成，默认情况下，key 只是按照生成顺序产生的一个累加数：

```js
1
2
3
```

我们可以认为 `key` 是一个静态的属性，通常，它是脱离于文档结构存在的，例如节点位置变化了，它仍然是不变的。

Slate 提供了一系列基于 key 的 API 给开发者，去访问或者操作节点：

```js
editor.moveNodeByKey()
editor.removeNodeByKey()
editor.replaceNodeByKey()
```

## Path

我们提到，`key` 是节点的静态索引，不依赖于文档结构而存在，因此，它**没有反映节点位置的能力**。假定我们要访问对应 key 的节点，就需要从根节点开始，逐层遍历文档树的各个节点，直到匹配为止。设想一下，当我们的文档内容非常长时，这样做的开销是非常大的。

故而 Slate 设计了 **Path（路径）** 这个模型去描述节点位置，它类似于一个**动态索引**。它是一个 Immutable List，也就是一个数组，Path `[i, j]` 它就表示了从根节点开始，先到达根节点子孙中的第 `i` 个节点 `Node_i`，再到达 `Node_i` 的第 `j` 个节点 `Node_i_j` 就是该路径对应的节点。

![Slate 寻址树]()

基于 Path， 我们能访问节点的祖先、兄弟以及子孙，Slate 为此提供了一个 Path Utils 来处理路径关系。

## 访问节点

现在，假定我们当前的节点树为：

![节点树]()

```js
   root
a   b   c
  d e f
   g h
```

那么节点 g 的路径就为：

```js
[1, 1, 0]
```

我们分别看看如何通过该路径路径获得其父节点、兄弟节点以及子孙节点。



