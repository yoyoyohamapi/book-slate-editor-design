# 工作流程

command -> operations -> apply operation to value -> dirty-paths -> normalize -> new value



Slate 是一个数据驱动的编辑器，每当数据模型发生变更，都会生成新的视图：

![view-to-model-to-view]()

而当用户在编辑器中执行了某个操作，生成的 UI Event 又会被编辑器所拦截，借此生成新的数据模型，用户最终看到自己的操作被正确响应。Slate 定义了 Controller 来负责这个过程：

![editor-controller-workflow]()

- 根据 Event 类型的不同，Controller 调用不同的指令（Command）
- Command 生成若干 Operation
  - Operation 被应用到（apply）当前编辑器模型，Controller 标志了受影响的节点路径为 dirty path
- Command 执行完毕，Controller 会根据搜集到的 dirty path，为对应的节点的做 normalize



## Comamnd



## Operation

