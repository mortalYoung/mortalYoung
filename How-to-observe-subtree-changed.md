# MutationObserver
在开发的过程中，我遇到了一个问题。在寻求解法的过程中，我学习了有关 `MutationObserver` 的相关知识，我在此记录下来。

以下内容整理自 MDN：
## 构造器
我们可以在 `React.Comonent` 组件的 `componentDidMount` 或者 `React.FC` 的 `useLayoutEffect` 中进行初始化操作.
```js
const observer = new MutationObserver(callback);
```
`callback` 即每次观察到变更则会触发的回调函数。

### callback
当你的 `callback` 如下所示的时候

```js
const callback = function (mutations, observer) {
    console.group();
    console.log("mutations:", mutations);
    console.log("observer:", observer);
    console.groupEnd();
};
```
那么，当每次变更的时候，控制台就会输出如下结果:
```js
▼ console.group
|    mutations: [MutationRecord]
|    observer: MutationObserver {}
```

### MutationRecord
第一个数据类型为 `MutationRecord[]`, 是一个 `MutationRecord` 的数组，内容包括
```typescript
type MutationRecordType = "attributes" | "characterData" | "childList";

interface MutationRecord {
    readonly addedNodes: NodeList;
    readonly attributeName: string | null;
    readonly attributeNamespace: string | null;
    readonly nextSibling: Node | null;
    readonly oldValue: string | null;
    readonly previousSibling: Node | null;
    readonly removedNodes: NodeList;
    readonly target: Node;
    readonly type: MutationRecordType;
}
```
> 引用自 [MutationRecord#properties](https://developer.mozilla.org/en-US/docs/Web/API/MutationRecord#properties)

## observe
当我们初始化构造器以后，还需要通过调用 `observe` 方法来执行监听.
```js
observer.observe(targetNode, observerOptions);
```
在 `react` 中使用，需要确保 `dom` 节点已经初始化。

其中 `observerOptions` 是监听的设置项
```ts
interface MutationObserverInit {
    /**
     * 如果不需要观察所有属性变化并且属性为真或省略，则设置为属性本地名称列表（不需要带命名空间）。
     * @examples ['style']
     */
    attributeFilter?: string[];
    /**
     * 如果属性为真或省略，并且需要记录变化前目标的属性值，则设置为 `true`。
     */
    attributeOldValue?: boolean;
    /**
     * 如果要观察到目标属性的变化，则设置为 true。如果指定了 attributeOldValue 或 attributeFilter，则可以省略。
     */
    attributes?: boolean;
    /**
     * 如果要观察到目标数据的变化，则设置为 true。如果指定了 characterDataOldValue，则可以省略。
     */
    characterData?: boolean;
    /**
     * 如果 characterData 设置为 true 或省略并且需要记录突变之前的目标数据，则设置为 true。
     */
    characterDataOldValue?: boolean;
    /**
     * 如果要观察到目标孩子节点的变化，则设置为 true。
     */
    childList?: boolean;
    /**
     * 如果不仅要观察到目标的变化，还要观察目标子树的变化，则设置为 true。
     */
    subtree?: boolean;
}
```

## DEMO 
如下是一个简单的 `MutationObserver`
```js
const observer = new MutationObserver((mutations, observer) => {
    console.group();
    console.log("mutations:", mutations);
    console.log("observer:", observer);
    console.groupEnd();
});
// 目标节点
const dom = document.getElementById("target");
observer.observe(dom, {
    characterData: true,
    subtree: 1,
    childList: true,
});
```
该 `demo` 实现了当 id 为 target 的节点及其子树发生变化的时候，会通过 `MutationObserver` 监听到变化后，在控制台输出变化详情.


## IDEA
1. 该方法可以实现对 `dom` 节点的监听, 那就可以实现观察者模式。

比如：当某处的数据结构为 `Array.map` 出的结构时，那就可以对 `dom` 是进行监听，在另一处进行 `update` 操作.

2. editable + MutationObserver ≈ textarea