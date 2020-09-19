# Memorize

在 Slate 中，存在一些高频调用的操作，一些调用的性能开销还比较大，为此 Slate 对这些方法做了 memorize。与 lodash 等库对函数做 memorize 的目标一致，都是当调用参数相同时，直接返回缓存结果，但 Slate 是对实例化对象的方法做的 memorize。

例如，Node Interface 就对节点实例的下面这些方法做了 memorize：

```js
memoize(NodeInterface.prototype, [
  'getFirstText',
  'getKeysToPathsTable',
  'getLastText',
  'getText',
  'normalize',
  'validate',
])
```

Slate 使用了 WeakMap 作为缓存的存储介质，以上面 Node Interface 的 memorize 为例，Slate 为每个 Node 被 memorize 方法生成的缓存结构为：

```js
{
  noArgs: {
    getText: 'Hello World',
  },
  hasArgs: {
    normalize: {
      Symbol('STORE_KEY'): WeakMap
    }
  },
}
```

即 Slate 将缓存缓存划分为无参数调用和有参数调用。若我们调用了方法 `node.normalize(editor)`，将会以此寻找：

```
const { hasArgs } = WeakMap.get(node)
const cached = hasArgs.normalize[Symbol('STORE_KEY')].get(editor)
```

来获得缓存。

Slate 使用 WeakMap 作为缓存的优势是：

- 使用对象作为节点 key
- 更好的访问性能
- 更好的内存利用：WeakMap 对垃圾回收更加友好，当作为 key 的对象引用不再被使用了，那么对应的缓存也可以被 GC 所清理

对于 GC 友好，我们可以做个实验：

```js
const wmap = new WeakMap();

const a = {}
wmap.set(a, 1);

function foo() {
  const b = {};
  wmap.set(b, 2);
}

foo();

console.log(wmap);
```

函数 `foo` 运行完成后，其作用域内的对象 `b` 会被回收，我们再打印当前的 WeakMap 对象，会发现只还有了对象 `a` 所映射的值。

## 参考资料

- [MDN-WeakMap](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakMap)