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

###
