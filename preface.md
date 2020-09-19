## 什么是 Web 富文本编辑器

富文本编辑器，指的是用户能够在浏览器中编排富文本（具有风格及排版的文本，如可以设定字体样式，进行图文混排等）。通常，在英文社区，更喜欢富文本编辑器称作是 ** WYSIWYG（what you see is what you get）**，即 “所见即所得编辑器”。

在富文本编辑器出现之前，浏览器已经具备了**展示**富文本的能力，开发者可以通过组织 HTML 和 CSS，实现对字号，字色等样式控制。但对于用户输入，所提供的 `<textarea />` 和 `<input />` 都只允许用户输入纯文本，能力十分单薄。

在 Web 浏览器中，实现对内容的**所见即所得**编辑，所谓的 **得（Get）**也就是指的 HTML 内容。因此，当我们在一个 WYSIWYG 编辑器中，多选中的文本进行加粗，为了看见文字变粗，就需要生成 `<b>xxx</b>` 这样的节点。亦即，Web WYSIWYG 编辑器是让用户能够直接在浏览器中**编辑 HTML **内容。

## contentEditable —— 让节点可编辑

早期的富文本编辑器实现中，需要自己手动实现光标、输入框等等，每次输入，编辑器的内部逻辑要能够动态绘制出当前产生的 DOM 内容，例如当用户在编辑器中执行，编辑器的内部此时就要创建一个新的段落（如生成一个 `<p />` 节点）。

而微软的 Internet Explorer 在 5.5 版本（于 2000 年 7 月发布）中引入了 contentEditable 特性，即内容可编辑性。这让 HTML 内容能在 IE 浏览器中就能被编辑。contentEditable 涉及两个属性 `designMode` 与 `contentEditable`，前者可以控制整个文档（document）是否可以编辑，后者则是控制某个元素（element）是否可编辑。当时，微软发布的这个特性，除了一个[简要的使用文档](<https://docs.microsoft.com/en-us/previous-versions/windows/internet-explorer/ie-developer/platform-apis/ms537837(v=vs.85)?redirectedfrom=MSDN>)，没有对可编辑性的内在行为和实现做更多描述。

> 虽然是 IE 5.5 引入了 contentEditable，但对应的功能早在 IE 4 就可以通过自定义 ActiveX 控件实现。

行为及实现没有规范，导致该特性只局限在了 IE 浏览器中使用。试想，当用户在可编辑内容中，敲入一个回车，这个时候，编辑器应当做什么？凡此种种，都需要一个规范来约束。例如，[WHATWG](https://www.wikiwand.com/en/WHATWG) 成员 Anne van Kesteren 在开始了对微软 contentEditable 特性的逆向工程之后，于 2005 年发表了一篇名为 [More on contenteditable](https://annevankesteren.nl/2005/07/more-contenteditable) 的博文，在其中举了一个例子：

```html
<!DOCTYPE html>
<div contenteditable>
  test
</div>
```

```html
<!DOCTYPE html>
<div contenteditable>
  <div>
    test
  </div>
</div>
```

当我们分别在这两个例子中敲入回车时，在 IE 5.5 中，前者将生成一个 `<p>` 元素，而后者生成的却是一个 `<div>`。

因此，为了推动 HTML 在浏览器中可编辑性的应用，WHATWG 小组开始着力于 contentEditable 的规范制定。同样是在 2005 年 7 月，Anne van Kesteren 撰写了[第一版 contentEditable 的规范](https://annevankesteren.nl/projects/whatwg/spec)。最终，经过标准委员会的不断努力，终于形成了 HTML 5 中的 [contentEditable 规范](https://html.spec.whatwg.org/multipage/interaction.html#contenteditable) 规范，这也是目前浏览器所遵循的规范。

在规范中，约定了两个角色：

- **editing host**：即正被编辑的 HTML 元素。如果某个 HTML 元素开启了 `contentEditable` 属性，那么这个元素就是一个 editing host；或者 `document` 开启了 `designMode`，那么整个 HTML 文档下的元素都是 editing host。

- **editable**：即可编辑元素。若 HTML 元素是 editing host 的子孙，那么它就可以被编辑，另外，可编辑元素的子孙也是可编辑的（除非这些子孙被声明了 `contentEditable` 为 false）。

## `execCommand` —— 使用命令进行编辑

通过 `contenteditable` 及 `designMode` 属性能让 HTML 内容能够被编辑，但是，它们仅仅是编辑能力的开关和入口，但如何根据用户行为动态设置 HTML 内容，IE 5.5 引入 contentEditable 特性时并没有考虑。直到 Firefox 3 问世，其不仅支持了 contentEditable，还配套了能够与可编辑元素进行互动的 API —— `document.execCommand`：即通过**指令（Command）**完成内容编辑。

例如，我们想要完成当前选中文本的加粗就只需要执行：

````js
document.execCommand('bold');
```

当然，`document.execCommand` 并非无所不能，甚至还会是文本编辑器的实现掣肘，不仅仅是支持的指令优先，就连同一个指令，各浏览器的实现都有可能不同。因此，很多编辑功能仍然需要开发者进行事件劫持等操作才能实现。

总结一下，要实现一个 Web 富文本编辑器，就需要：

1. 可编辑性：开启 `contentEditable` 属性让编辑器内部的 HTML 内容可被编辑。
2. 编辑接口：使用 `document.execCommand` 编辑内容。

## 纷繁缭乱的编辑器

如此看来，既然浏览器已经提供了 `contentEditable` 及 `execCommand` 指令，我们似乎不需要再造 Web 富文本编辑器了，但社区中还存在各式各样基于二者实现的富文本编辑器，如 Draft.js、Slate.js、以及资历更老的 TicyMCE 和 CKEditor 等等，这是为什么呢？我们不妨思考下面几个问题？

### 功能够用了？

加粗、复制有了，但是我们想要在能够在可编辑节点中，创建一个标题呢？我们想要插入代码块呢？这些已经超过了浏览器原生的提供的 contenteditable 能力，因此，我们需要编辑器，是寄希望于编辑器本身具备了该功能，或者编辑器具备了可编程能力，让我们基于其内核，扩展编辑能力。

### 如何应对不同的浏览器及浏览器版本？

不同的浏览器，或者不同的浏览器版本，他们看待段落的方式可能不同，是 `<div>` 还是 `<p>`，加粗应当是 `<br>` 还是。亦或，我们在高版本浏览器上创建的内容，如何让它也能正确渲染到低版本浏览器。二逼更加纷繁缭乱的语言，以及对应的展示方式和输入处理

### 如何支持不同的语言？

比浏览器更加纷繁复杂的是世界上多达 7000 多种的语言，如何处理不同语言的展示、输入法也是富文本编辑器所面临的一大问题。

### 如何让我们的富文本跨平台？

浏览器并不是展示 UI 的唯一入口了，如果我们想要将富文本展示在 Android、IOS 或者各种小程序容器内，使用 HTML 描述富文本内容就成为了一种局限。

### 如何让 Word 能够粘贴到富文本？

假定我们复制了一段 Word 或者 Markdown 等不同数据源的内容到剪贴板，是否可以让这些内容粘贴到我们的富文本时，其格式还尽可能保留呢？

### 如何支持协同编辑？

支持多人协同办公已经成为了现代 Web 应用的一个特征了，如果要实现多人在线编辑同一个文档，那么就绕不开去对编辑者行为做记录（operation），记录之间的差异做比较（diff），以及处理记录合并时的冲突（conflicts），那么 HTML 也不是这个过程的最好描述方式。

因此，富文本编辑器这几年仍不断涌现，它们都致力于通过更好的结构设计、更优雅的编程接口，以及对浏览器和多语言更深刻的认识，来解决或者提供：

- **基础编辑能力封装**：封装诸如字体设置、图片等常用行为，让绝大部分开发者享受到 “开箱即用”。
- **浏览器兼容性**：编辑器指令的实现，对于输入行为的管控，在各个浏览器中的实现也是有差异的，兼容性优异的富文本编辑器能让开发者远离泥沼，专注于编写功能。
- **扩展体系**：富文本编辑器并不能定义所有功能，因此它需要要提供扩展体系调动开发者的智慧，让开发者能方便地定义自己需要的功能。
- **跨平台**： 随着展示用户界面的 UI 设备和方式越来越多，富文本编辑器也要能够跨平台，或者提供了让开发者实现跨平台的手段。
- **多语言支持**：优秀的编辑器还需要考虑到语言阅读顺序的差异，输入法差异。
- **多格式支持**：不同的粘贴源，是否能按照用户预期被粘贴到当前的编辑器中。
- ......

接下来，我们就将以 Slate.js 为编辑器范本，了解 Web 富文本编辑器的设计和实现，但不必沉溺于某些细节和边界处理，我们应当从 Slate.js 的设计中了解富文本编辑器的构成与考量，从其实现中深化对富文本编辑器开发需要留意的方方面面。

## 参考资料

- [The WHATWG Blog](https://blog.whatwg.org/the-road-to-html-5-contenteditable)