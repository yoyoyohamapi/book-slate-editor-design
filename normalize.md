# Normalize

在 [编辑器模型]() 一节中，我们介绍过，HTML Element 可以通过 `normalize` 方法实现节点的标准化，例如合并多个文本节点：

```js
const element = document.createElement('div');

element.appendChild(document.createTextNode('1 '));
element.appendChild(document.createTextNode('2 '));
element.appendChild(document.createTextNode('3 '));

element.childNodes;  // NodeList [text, text, text]
element.textContent; // 1 2 3 

element.normalize();
element.childNodes;  // NodeList [text]
element.textContent; // 1 2 3
```

