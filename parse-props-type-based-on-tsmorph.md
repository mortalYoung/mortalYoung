# 基于 ts-morph 解析组件的参数类型

## 序言

- 由于最近接触了比较多的有关解析 AST 的需求, 然后在公司内部项目中发现基于 `ts-morph` 来解析貌似看起来不错, 但是在 google 上又搜不到 tutorials, 所以有了这篇文章
- 主要目的是**为了记一下解题思路**
- 本文是简单使用，如果想要**深入了解，请自我学习，看别人的文档永远只是管中窥豹**
- 如果你和我有不同的意见，**欢迎提 issues 给我**

## `ts-morph`

- 要用到的工具有 `ts-morph` 包, [官网文档](https://ts-morph.com/)是有的，但是不够详细，且没有中文翻译。
- 只有在使用的过程中才会慢慢体会到这个包到底是什么用的

> Setup, navigation, and manipulation of the TypeScript AST can be a challenge. This library wraps the TypeScript compiler API so it's simple.

官网文档中给出的这句话第一遍看得让人云里雾里，多读几遍再配合上实践才能看懂其中深意。

**个人理解来说**，我认为 `ts-morph` 这个包提供的功能是：

1. 支持基于 TypeScript compiler API 解析已存在的 ts 文件
2. 支持基于 TypeScript compiler API 生成 ts 文件
3. 支持基于 TypeScript compiler API 修改已存在的 ts 文件

> **注**：以上的 ts 文件包括 tsx 文件

## 解题思路

### 准备工作

- 首先我们需要明确的目标是**我们要基于 `ts-morph` 来实现解析组件的参数**
- 其次，我们需要知道组件分为函数组件和类组件, 而每种组件的 export 方式也不尽相同，至少有两种以上的写法，包括 ` 直接 export` 和 ` 间接 export`

以函数组件为例

```ts
// 直接 export
export default FunctionComponent = ()=> {};
// 间接 export
const FunctionComponent = ()=> {};
export default FunctionComponent;
```

这两种写法虽然在使用上没有区别，但是解析的时候会略有区别，所以在这里先提一下

### Code

首先项目中 `install ts-morph`, 并新建一个 `__test__` 目录来存放我要测试用到的实例

然后新建 `index.js` 文件存放实例代码，并在 `__test__` 中新建了 4 种写法组件

目录结构如下：

```
├── package.json
├── __test__
│   ├── assignments.tsx # 函数组件 - 间接 export
│   ├── index.tsx # 函数组件 - 直接 export
│   ├── reactComponent.tsx # Class 组件 - 直接 export
│   └── classAssignments.tsx  # Class 组件 - 间接 export
└── index.js
```

接下来，我只需要在 `index.js` 中解析就可以了

```js
const {Project} = require("ts-morph");

const project = new Project();
// 添加文件到 project 这个对象中去
project.addSourceFilesAtPaths("__test__/*.tsx");
```

读取 `__test__` 下全部的文件都加入到 project 这个对象中，然后我们先用 `__test__/index.tsx` 这个 demo 来测试是否能成功读取参数类型

```js
// 源文件
const sourceFile = project.getSourceFileOrThrow("index.tsx");
// 获取 export 声明的语句
const exportAssignments = sourceFile.getExportAssignments();
```

然后从 project 对象中获取源文件，再通过获取源文件的 getExportAssignments 方法来获取到当前 `index.tsx` 文件中全部的 export assignments 的语句

> **注**：这里获取到的 `exportAssignments` 这个参数有可能为空数组，即文件中不存在 export assignments 的语句，比如 Class 组件 - 直接 export 就不存在 exportAssignments

```js
// 获取到 export default 的那条声明语句
const defaultExport = exportAssignments.find((i) => !i.isExportEquals());
// 先通过获取 表达式 再取类型
const type = defaultExport.getExpression().getType();
```

由于我们拿 `index.tsx` 来做测试，所以 `exportAssignments` 不存在空的情况，所以直接继续执行了。如果是 Class 组件 - 直接 export 的方式，需要去判断该值是否为空

> **注**: 这里拿到的类型是整个组件的类型 而不是 props 的类型, 即类似于 `(props:TestProps) => JSX.Element` 这种类型

接下来需要区分 class 组件和函数组件，通过拿到的类型提供的 `isClass` 方法来区分

```js
// 判断是否为 class 组件
const isClass = type.isClass();
```

由于是 `index.tsx` 是函数组件，所以 `isClass` 此时是 `false`，那么代码进入到 `else` 流程里

`type.getCallSignatures` 这条语句是为了获取到函数组件的签名

对于大多数熟知的语言来说，一个函数是存在重载，继承等情况的，但是对于 js 是不存在，且组件应该也是不存在重载的

所以这里应该只有一个

```js
// 获取函数的签名，由于 js 语言的特殊性，所以此处的签名应该只有一个
const callsignatures = type.getCallSignatures()[0];
// 获取到参数的声明的类型,
// 可能存在多个的情况 所以 propsType 是数组
const propsType = callsignatures
  .getParameters().map((p) => p.getValueDeclaration().getType());

propsType.forEach((pType) => {
  // 解析类型
  return mockTransformType(pType);
});
// 完成
```

> 函数的方法名 + 形参列表，就叫做该函数的方法签名。

通过获取签名再获取参数再获取参数声明的类型就可以了

> `mockTransformType` 该方法无实际意义 是个 `void Function`, 解析过程在获取到 `propsType` 的时候就结束了

---

以上就完成了对**简单的函数组件**的参数类型解析, 接下来解析**简单的类函数**

---

思路重新回到 `isClass` 这里，刚才上面是进入到 `else` 流程里去

现在假如是类组件的话，这里就该是 `true` 了，那么解析来就要进入 `if` 流程里

```js
// 获取 class 组件的名称
const className = defaultExport.getExpression().getText();
// 通过名称获取目标 class
const targetClass = sourceFile.getClass(className);
// 获取继承的类型的参数
// 通常的， React.Component 和 React.PureComponent 的 props 在前 state 在后
// 所以这里拿数组的第一个元素
const propsType = targetClass.getExtends().getTypeArguments()[0].getType();

// 解析类型
mockTransformType(propsType);
```

这里通过获取默认导出的那一行语句的表达的文本来获取到类组件的名称

> **注**：该情况只适用于 Class 组件 - 间接 export

然后通过 `getClass` 获取到类，再通过 `getExtends` 获取到继承的类型，再通过 `getTypeArguments` 获取到继承类的类型参数

由于通常来说 `React.Component 和 React.PureComponent` 的类型参数 `class React.Component<P = {}, S = {}, SS = any>` 我们可以看到 Props 在前，State 在后

所以这里我就直接拿数组的第一项了，第二项是 state

然后再获取 `Type` 就可以了

---

还没完，上文提到了这种方法只适用于 Class 组件 - 间接 export ，我们还有 Class 组件 - 直接 export 需要解析

---

思路回到 `exportAssignments` 变量声明的地方，我们提到过当组件为 Class 组件 - 直接 export 方式的时候获取到的该变量为空数组，那我就借助这个条件来区分是否为 Class 组件 - 直接 export，并对该方式的组件单独处理

```js
if (!exportAssignments.length) {
  // 表明不是用声明式写的组件 那就是 直接 export default Component 的这种形式
  const defaultclass = sourceFile.getClasses().find((i) => i.isDefaultExport());
  const propsType = defaultclass.getExtends().getTypeArguments()[0].getType();
  // 解析类型
  mockTransformType(propsType);
}else{...}
```

代码一看就懂就不解释了

## 最后

再次重申，只是对于简单组件的参数解析，如果你要问我 HOC 怎么解析，获取继承于 React.FC 的怎么解析，那我觉得你需要自己深入学习。**授人以鱼不如授人以渔**

[文档在此，全靠自己](https://ts-morph.com/)

另外：文中提到的代码都在 👉[这个仓库](https://github.com/mortalYoung/ts-morph-demo)
以上。
