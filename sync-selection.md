# 选区同步

除了需要同步 DOM 节点与数据模型上的节点对象，Slate 还需要完成选区的同步。对于 contenteditble 节点，可以通过监听 `` 事件来获得当前在节点中的选区。



This adds support for the native `selectionchange` event, which prevents people from having their selection blocks by concurrent changes being applied to the editor.



但是，这个还不够，编辑器只是页面内容的一个部分，如果



## 参考资料

- [Use native selectionchange event as opposed to React's onSelect](https://github.com/ianstormtaylor/slate/issues/1135) 