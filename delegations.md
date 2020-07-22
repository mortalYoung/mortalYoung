# Event Delegations

Event Delegations => 事件委托

## When To Use It

- 当你有大量节点需要绑定事件的时候, 不必每一个节点都绑定上事件
- 给全部的节点的父级节点绑定事件, 并通过 event.target 来判断
- 这就是 事件委托


## difference between e.target and e.currentTarget

use following DOM for example

```js
<ul>
  <li></li>
  <li></li>
  ...
  <li></li>
</ul>
```

when I `addEventListener` on **UL**, and absolutely I will distinguish each LI by `e.target`.

Apparently, **`e.target` will gets the true event trigger**. `e.target` 会找到真正的事件触发者, 本例子中即为 `LI` 节点

As for `e.currentTarget`, **It only gets the event binding** `e.currentTarget` 只会寻找当前事件的绑定者, 本例子中即为 `UL` 节点
