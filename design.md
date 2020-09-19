编辑器模型

假定我们在最高的高空，可能看到的编辑器就是下面这个形态：

![含有选取与节点信息的图]()

它由两个部分组成：

- __节点（Node）__：节点容纳了我们能看到的富文本内容，富文本容器也是一个节点，容纳了其他节点
- __选区（Range）__：当前选中的区域，如果区域的起点和终点重合，那看到就是一个光标

# 节点

## 节点类型

在 HTML 现行规范中，[节点（Node）](https://dom.spec.whatwg.org/#interface-node)被定义为一个高级接口，这样，不同类型的节点只要继承自该接口，就能使用同样的 API 访问以及修改节点。

对于一个网页来说，继承自 Node 的 [Document](https://developer.mozilla.org/zh-CN/docs/Web/API/Document) 即是其内容的入口，它是一棵 DOM 树：

![DOM tree]()

在这颗 DOM 树中，挂载了不同类型的、实现了 Node 接口的节点，更具体的，它们实现了 [Element 接口](https://developer.mozilla.org/zh-CN/docs/Web/API/Element) ，因此，具备了诸如描述样式和尺寸的能力：

```js
Element.classList
Element.styles
Element.clientHeight
Element.getBoundingClientRect()
Element.getComputedStyle()
```

我们常见的 `div`、`p`、`span` 等 HTML 元素是继承自 [HTMLElement](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement)（它继承自 Element）。在 HTML 5 之前，HTMLElement 被分为了:

- **块级元素(Block Level Element)**：块级元素占据了父容器的整个空间，视觉上就是形成了一个 “块”。文档中每新增一个块，首先要新增一个容纳这个块的行。
- **行内元素(Inline Level Element)**：行内元素只占据了内容所需要的空间（标签边框所包括的内容）。

![块级元素包含了行内元素]()

而在 HTML 5 之后，HTMLElement 的按照[内容类别](https://developer.mozilla.org/en-US/docs/Web/Guide/HTML/Content_categories)进行了细化：

![HTML Content Category]()

另外，在 HTML 中，还引入了 **Void Element** 的概念，Void Element 即空节点，空节点不允许包含任何的内容。例如 `<input>`、`<link>` 等元素都是 Void Element，下面这些定义因此也都是无效的：

```html
<p>
  <input value="12345">Content</input>
  <img src="https://this-is-an-image.jpg">Image</img>
</p>
```

## 节点关系

一篇 HTML 文档，可以被一棵 DOM 树所标识，一个节点也是一个 DOM 子树，因此，其关系网就存在：祖先、孩子、兄弟。HTML 规范中规定了使用下属属性来访问节点关系：

- `Node.parentNode`：访问节点父节点
- `Node.childNodes`：访问节点子孙
- `Node.previousSibling `与`Node.nextSibling`：访问节点的兄弟节点

节点间关系的定义能帮助我们方便的遍历和定位节点。

## 节点内容

在 HTML 规范中，定义了 [Text 接口 ](https://dom.spec.whatwg.org/#text) 用来表示节点文本内容，它继承自 [CharacterData](https://developer.mozilla.org/zh-CN/docs/Web/API/CharacterData) ，后者用于表示 Node 所包含的字符。

我们可以通过 `Document.createTextNode()` 向节点中插入多个 Text，并通过 `Node.textContent` 访问节点的文本：

```js
const element = document.createElement('div');

element.appendChild('1 ');
element.appendChild('2 ');
element.appendChild('3 ');

element.childNodes; // NodeList [text, text, text]
element.textContent; // 1 2 3 
```

在上面这个例子中，还可以通过 `Node.normalize()` 将 node 结构规范化，让相邻的文本节点合并为一个文本节点：

```js
element.normalize();

element.childNodes; // NodeList [text]
element.textContent; // 1 2 3 
```

## 节点数据

除了节点文本以外，我们往往还需要为节点绑定数据，例如，对于一个图片节点，我们需要设置图片源和图片尺寸，HTML `img` tag 为此提供了 `src` 、`width`、`height` 等属性供我们进行图片数据的绑定。更一般地，我们可以通过 `data-*` 设置节点的数据：

```html
<pre
  id="code-block"
  data-syntax="javascript"
  data-theme="dracula"
/>
```

如此绑定的数据，可以通过 `HTMLElement.dataset` 进行访问：

```js
const code = document.querySelector("code-block");

code.dataset.syntax; // javascript
code.dataset.theme; // dracula
```

# 选区

## Selection

选区（Selection）表示的是当前用户选中的内容：

![一个选区]()

从上图中，我们可以看到，一个选取需要包含如下：

- 选区的起点：我们需要知道选区**起始于哪个节点**下，在**该节点中需要偏移多少**
- 选区的终点：我们需要知道选区**结束于哪个节点**下，在**该节点中需要偏移**
- 选区的方向：用户既可以向前选择，也可以向后选择
- 偏移量：除了知道选区的起点、终点落在哪个节点下，还需要知道在该节点下，选区具体需要偏移多少。

因此，在 HTML 中，一个选区 `Selection` 对象被设计了如下属性：

- `anchorNode`：选区的起始节点
- `anchorOffset`：选区起点在节点中的偏移
- `focusNode`：选区终点
- `focusOffset`：选区终点在节点中的偏移
- `isCollpased`：选区是否折叠

选区的方向则可以通过 anchor、focus 的位置判定，若 focus 落在 anchor 之后，则为向后选择，若落在 anchor 之前，则为向前选择，二者重叠时，即选区被折叠。

## Range

那么如何表示选区内部选中的内容呢？HTML 标准还定义了 [Range](https://developer.mozilla.org/en-US/docs/Web/API/Range)，一个 `Range` 对象描述了一个包含若干节点和节点文本的文档片段。一个文档片段同样需要确定起点和终点，因此，HTML 为 Range 对象定义了类似于 Selection 对象的属性：

- `collapsed`：选区是否被折叠
- `commonAncestorContainer`：选区起点、终点公共的父容器
- `startContainer`：选区起点所在的节点
- `startOffset`：选区起点在节点中偏移
- `endContainer`：选区终点所在的节点
- `endOffset`：选区终点在节点中的偏移

Selection 最早是作为 Netscape 浏览器的特性被引入，之后又被 Firefox 的 Gecko 所实现，最后接着被其他浏览器所实现。Netscape 将 Selection 实现为了由多个 Range 所构成，这也合乎语义，一个选区应当可以包含多个文档片段。这样的设计，能让用户同时选中一个表格的若干列，但是却为开发者，甚至是同样实现了 multiple range 的 Gecko 的开发者遇到了难以应付的边界场景。

因此，当其他浏览器在实现 Selection 时，不再允许一个 Selection 包含多个 Range，但是，为了尽可能兼容 API，还是为 Selection 支持了 `removeRange()`、`getRangeAt()` 等方法，只是开发者在使用这些 API 时，一般都只能传入 0 作为 index，即一个 Selection 对象只含有一个 Range 对象。

# Mirror the DOM

Slate.js 的一大设计目的就是让编辑器的数据与 UI 分离，在数据模型上的设计上，则是尊崇了 “Mirror the DOM” 的理念，尽可能按照现行的 DOM 标准去抽象数据模型。这种亲近标准的理念让开发者不用再被额外灌输新的知识，只要熟悉前端，也就能轻松地操控 Slate 的数据模型。

既然要 Mirror the DOM，Slate.js 在数据模型的设计上，也就必须包含**节点**和**选区**模型的设计，接下来，我们看看 Slate.js 是如何设计节点和选区的。

## 对节点的模拟

Slate 在模拟 DOM 时，采用的 Model + Interface 的模式，Model 定义了数据结构和数据的构造方式，Interface 实现了功能。

![]()

### Node

![Node Model & Node Interface]()

在节点的设计上，类似于 DOM，Slate.js 同样使用 Node 作为最高级抽象，上文中，我们提到，在设计节点时，要考虑这四个位面：

- **如何划分节点类型**
- **如何处理节点关系**
- **如何表示节点内容**
- **如何绑定节点数据**

一个 **Node Model** 含有如下属性：

- `key:string`：节点在当前 Slate 实例中的索引
- `data: Immutable.Map`：data 保存了节点数据
- `nodes: Immuable.List<Node>`：当前节点的子孙
- `object: string`：类型
- `text: string`：这是一个计算属性，用来获得当前节点下的文本内容

除了属性约束，Node Model 还提供了创建节点和节点序列等构造模型的静态方法：

```js
class Node {
  static create(attrs = {}) {}
  static createList(elements = []) {}
  static createProperties(attrs = {}) {}
  static fromJSON(value) {}
}
```

而在 **Node Interface** 中，提供了了节点遍历、节点内容访问、节点规范化等能力：

```js
class NodeInterface {
  getText() {}
  getNode(path) {}
  getPath(key) {}
  normalize() {}
  validate() {}
}
```

## Element

类似于 HTML Element，Slate 将节点类型分为：

- **Document Element**：表示编辑器中节点树的根节点
- **Block Element**: 表示编辑器中的块级元素，如段落、表格等
- **Inline Element**：表示了编辑器中的行内元素，如链接、图片等
- **Text Element**：表示了编辑器中的文本节点

不同类型的节点的 Model 设计都与 Node Model 类型，只是分别实现了 Model 的创建方法和序列化到 JSON 的方法：

```js
class Document {
  static create(attrs = {}) {}
  static fromJSON(object) {}
  static toJSON(options = {}) {}
}
```

而在 Element Interface 中，提供了对于访问父亲和相邻接点、添加格式化内容、创建及插入子节点等能力：

```js
class ElementInterface {
  addMark(path, mark) {}
  
  getParent(path) {};
  getPreviousNode(path) {};
  getNextNode(path) {};
  findDescendant(predicate = identity) {};
  
  insertNode(path, node) {}
  removeNode(path) {}
  
  // ...
}
```

与 DOM 不同的是，Slate 使用了路径（path）来构建节点间关系，path 描述了**从根节点开始，访问到某个节点要经过的路径**，这在 [节点寻址]() 一节中将会有更详细的介绍。

## 对选区的模拟

### Range

而在选区的模拟上，Slate 融合了 DOM Selection API 与 DOM Range API 的设计，设计了 **Range** 作为选区的最高级的抽象：

![]()

Range 也采用 `anchor`、`focus` 来描述一个片段的起终点，但更进一步，Slate 还涉及了 **Point** 来描述坐标，其完整的属性设计为：

- `anchor`：当前 range 的起点
- `focus`：当前 focus 的终点

另外，根据当前 range 的状态，还返回了如下计算属性：

- `end` 与 `start`：如果说 `anchor`/`focus` 是 range 的事实起终点，那么 `start`/`end` 则是 range 的视觉起/终点，`start` 总在 `end` 之前（或者二者重合）
- `isBackward` 与 `isForward`：选区方向是向前还是向后
- `isCollapsed` 与 `isExpanded`：是否折叠
- `isSet` 与 `isUnset`：起点终点是否均被设置

### Point

在 DOM 中，我们的 anchor、focus 能够直接指向 DOM 节点，Slate 脱离的 UI 设计的数据模型就需要有新的数据结构来表明 anchor、focus 的指向，因此，它设计了 Point 来描述 Slate 中的坐标信息：

![]()

一个 Point 对象具有如下属性：

- `key`：当前位置上的 Text 节点的 key
- `path`：当前位置上的 Text 节点的寻址路径
- `offset`：当前位置在对应 Text 节点上偏移了多少字符

### Selection

Selection 继承自 Range，用来描述当前用户选区，除了继承了 Range 的属性外，还提供了这些属性：

- `isFocused` 与 `isBlurred`:  当前编辑器的失焦与聚焦情况
- `marks`：当前选中文本所包含的格式化 mark 信息







## 参考资料 

- [W3C Working Draft - Selection API](https://www.w3.org/TR/selection-api/#background)
- [DOM Living Standard](https://dom.spec.whatwg.org/#interface-node)



