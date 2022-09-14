# Why

There were several complex react components to begin with, but now we have to refactor it. Because marjority of users know little of front-end, we should allow back-end user to define the layout.


### First

Fisrt, we should abstract react component into data. And it's noticed that we should consider it as a solo component which means we shouldn't consider the relationship between components at first.


For example, now we have a input component in page. And now we should render it by engine with a specific data.

```
[a specific data] ------[engine]------> [input component]
```

We should define the data's structure. 
- `type` is required for defining the type of component. 
- `title` is required for defining the label of component.
- `name` is required for defining the key of component which be used for passing to back-end.
- `required` is optional for defining the required about component. It's defined in Form.
- `rules` `initialValue` `noStyle` are optional. It's defined in Form.


And now based on these field, we should acheive the engine.

[omitted.]

### Then

And then we could create components based on specific data like the following:
```js
const data = {
    type: 'number',
    title: 'title',
    name: 'sourceId,
}
```

But, that's not enough. We need `widget` to allow user to specify the component. Besides, we still need component key for ensure re-render component each time.

### Finally

We define `bind` and `depends` to make relationship between components.

For more details, see [Taier-ui](https://github.com/DTStack/Taier/blob/feat_1.3/taier-ui/src/pages/editor/dataSync/index.tsx)
