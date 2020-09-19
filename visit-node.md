节点寻址

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

## path

我们提到，`key` 是节点的静态索引，不依赖于文档结构而存在，因此，它**没有反映节点位置的能力**。假定我们要访问对应 key 的节点，就需要从根节点开始，逐层遍历文档树的各个节点，直到匹配为止。设想一下，当我们的文档内容非常长时，这样做的开销是非常大的。

故而 Slate 设计了 **Path（路径）** 这个模型去描述节点位置，它类似于一个**动态索引**，当节点的在文档树中的位置发生了变化，其 path 属性也会变化。它是一个 Immutable List，也就是一个数组，Path `[i, j]` 它就表示了从根节点开始，先到达根节点子孙中的第 `i` 个节点 `Node_i`，再到达 `Node_i` 的第 `j` 个节点 `Node_i_j` 就是该路径对应的节点。

![Slate 寻址树]()

基于 Path， 我们能访问节点的祖先、兄弟以及子孙，Slate 为此提供了一个 Path Utils 来处理路径关系。

## 访问节点

假定我们当前的节点树为：

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

## 访问祖先节点

由于路径是按照节点在节点树中的深度创建的，沿着路径回溯（倒序访问路径），即可获得节点的祖先，例如 g 的父亲节点 e 的路径就是：

```js
[1, 1]
```

e 的父亲节点 a  的路径就是：

```js
[1]
```

假定我们要访问节点的第 `n` 个祖先，那么这个祖先的路径很容易推导：

```js
const ancestor = path.slice(0, -1 * n)
```

 `PathUtils.lift(path, n = 1)` 就完成了这样的路径回溯，而 Element Interface 则提供了 `getParent(path)` 、`getAncestors(path)` 来获得节点的祖先：

```js
const ancestors = document.getAncestors(path)
const parent = document.getParent(path)
```

> 由于 Slate 中节点 Path 都为相对于根节点 document 的路径，因此，我们多使用 `document.xxx(path)` 来寻址节点和节点关系。

## 访问兄弟节点

要获得相邻兄弟节点的路径，我们还要确定兄弟节点是左相邻还是右相邻。由于 Path 的深度反映了层级，兄弟节点与当前节点位于同层，因此，假定我们寻找其右相邻的第 `n` 个节点，就是在 Path 尾部向上递增 `n` 个数 ：

```
[1, 1, 1 + n]
```

使用 Immutable.List 实现就是

```js
const nextSibling = path.update(path.size - 1, index => index + n)
```

同理，访问左相邻节点就是：

```js
const previousSibling = path.update(path.size - 1, index => index - n);
```

`PathUtils.increment(path, n, index)`/`PathUtils.decrement(path, n, index)` 即完成了对路径某个节点的递增/递减，而 Element Interface 提供了 `getPreviousSibling(path)` 、 `getNextSibling(path)` 和 `siblings(options)` 、 来访问节点的兄弟:

```js
const previousNode = document.getPreviousSibling(path);
const nextNode = document.getNextSibling(path);
const siblings = document.getSiblings(path, { direction: 'backward' });
```

## 访问子孙节点

访问当前节点的子孙节点，在 Element Interface 中，提供了 `getDescendant(path)` 用于访问子孙节点：

```js
class ElementInterface {
  getDescendant(path) {
    path = this.resolvePath(path)

    if (!path || !path.size) {
      return null
    }

    let node = this

    path.forEach(index => {
      node = node.getIn(['nodes', index])
      return !!node
    })

    return node
  }
}
```

## key 与 path 的转换

由于 `path` 是节点的动态索引，会随着节点位置的变化而变化，因此，在 Node Interface 中，提供了一个 `getPath(key)` 来根据 key 获得当前节点的路径：

```js
class Node {
  getPath(key) {
      // ...
      const dict = this.getKeysToPathsTable()
      const path = dict[key]
      return path ? List(path) : null
  }

  getKeysToPathsTable() {
    const ret = {
      [this.key]: [],
    }

    if (this.nodes) {
      this.nodes.forEach((node, i) => {
        const nested = node.getKeysToPathsTable()

        for (const key in nested) {
          const path = nested[key]

          ret[key] = [i, ...path]
        }
      })
    }

    return ret
  }
}
```

Slate **自顶向下**建立了一份 key 与 path 的映射关系，假定我们当前文档的节点树为，并假定节点的 key 与节点名一致：

```js
       document
        a b c
          d
```

并且此时 key 与 path 的映射表为空，我们试图访问根节点的路径 `document.getPath('document')`，将分别调用：

```js
document.getKeysToPathsTable()  
  => a.getKeysToPathsTable()    // {a: []}
  => b.getKeysToPathsTable()    // {b: [], d: [0]}
     => d.getKeysToPathsTable()   // {d: []}
  => c.getKeysToPathsTable()    // {c: []}
```

最终，获得了一份根节点（document）下的 key 与 path 的映射表：

```js
{
  document: [],
  a: [0],
  b: [1],
  c: [2],
  d: [1, 0]
}
```

无论是访问节点还是操纵节点，都是依赖节点路径的，因此，Slate 对节点 `getKeysToPathsTable` 做了 memorize，当在同一个节点下查询 key 对应的 path 时，能跳过遍历，返回缓存中的结果。但是当文档规模变大后，其含有的 key 数量也会随之膨胀，缓存命中率将降低。

