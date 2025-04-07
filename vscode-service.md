# vscode-service

首先，实现通过 js 创建 html 界面，会有如下代码：

```js
const header = document.createElement('header');
const main = document.createElement('main');
const sidebar = document.createElement('sidebar');
const content = document.createElement('content');
main.append(sidebar, content);
const footer = document.createElement('footer');

document.body.append(header, main, footer)
```

上述代码会形成如下 html 结构

```html
<body>
  <header></header>
  <main>
    <sidebar></sidebar>
    <content></content>
  </main>
  <footer></footer>
</body>
```

## java-style

接下来，要改造成 java 风格的写法。于是乎，有如下代码：

```js
Class Header extends Element {}
Class Footer extends Element {}
Class Main extends Element {}

Class Container {
  constructor(){
    this.header = new Header();
    this.footer = new Footer();
    this.main = new Main();

    document.body.append(this.header.ele, this.main.ele, this.footer.ele);
  }
}

new Container();
```

## IOC

在实际代码中，模块之间必存在关联性。

```diff
...
- this.footer = new Footer();
- this.main = new Main();
+ this.footer = new Footer(header);
+ this.main = new Main(header, footer);
...
```
但是这种方式不够自动化，所以借助 IOC 来实现。

```ts
// header.ts
export const IHeaderService = createDecorator<IHeaderService>('header');
export interface IHeaderService {}
// headerService.ts
export class HeaderService implements IHeaderService {}

// footer.ts
export const IFooterService = createDecorator<IFooterService>('footer');
export interface IFooterService {}
// footerService.ts
export class FooterService implements IFooterService {
  constructor(@IHeaderService private readonly _headerService: IHeaderService){}
}

// standaloneService.ts
registerSingleton(IHeaderService, HeaderService);
registerSingleton(IFooterService, FooterService);
class StandaloneEditor {
  constructor(@IHeaderService private readonly _headerService: IHeaderService, @IFooterService private readonly _footerService: IFooterService){}
}

function create(){
  // do something...
}
```

由于要借助 IOC，所以就不能通过 new 的方式来创建实例。（或者说，其实是需要将 new 的代码藏起来，从表面上看不需要而已）

这里，我们把所有的功能点都理解为一个 Service（从代码角度来说，就是一个或者多个类），声明 Service 的核心逻辑为 xxxService 中的 class 定义。

但是需要注意的是，为了实现 IOC，这里我们需要抽一个类型出来。这个类型是负责定义 service 中实现的方法，它既可以用来给 class 做一个实现，也可以给其他 service 中的 IOC 做类型提示。

同时，这里借助 `createDecorator` 实现 IOC 所需要的装饰器。

### decorator

在搜索 TypeScript 实现 decorators 中关于 [parameter-decorator](https://www.typescriptlang.org/docs/handbook/decorators.html#parameter-decorators) 的章节中，我们可以看到创建一个参数装饰器的 demo 代码如下：

```ts
import "reflect-metadata";
const requiredMetadataKey = Symbol("required");
 
function required(target: Object, propertyKey: string | symbol, parameterIndex: number) {
  let existingRequiredParameters: number[] = Reflect.getOwnMetadata(requiredMetadataKey, target, propertyKey) || [];
  existingRequiredParameters.push(parameterIndex);
  Reflect.defineMetadata( requiredMetadataKey, existingRequiredParameters, target, propertyKey);
}
```

上述代码的意思是，获取 target 的 requiredMetadataKey 属性（是一个数组），然后把当前字段的 index push 进去，然后赋值到 requiredMetadataKey 属性上。可以用简单的如下代码表示：

> [!NOTE]
> 只是简单的理解，并不相同，自行去理解 reflect-metadata 相关的实现

```ts
const symbolKey = Symbol("required");

function required(target: Object, key: string | symbol, index: number) {
  let a: number[] = target[symbolKey[key]] || [];
  a.push(index);
  target[symbolKey[key]] = a;
}
```

在 VSCode 中存在类似的实现：
```ts
const serviceIds = new Map();

const DI_DEPENDENCIES  = '$di$dependencies';

export function createDecorator(serviceId: string) {
  if(serviceIds.has(serviceId)) return serviceIds.get(serviceId);
  const id = function(target: Object, key: string | symbol, index: number) {
    target[DI_DEPENDENCIES] ??= [];
    target[DI_DEPENDENCIES].push({ id, index });
  }
  id.toString = () => serviceId;
  serviceIds.set(id, serviceId);
  return id;
}
```


### registerSingleton

简单来说，逻辑如下：

```ts
const _registry = [];
function registerSingleton(id, ctor){
  _registry.push([id, ctor]);
}
```

然后在实例初始化的时候，进行如下的逻辑：

```ts
function create(){
  // 复杂的，以及防止死循环的遍历逻辑，获取 StandaloneEditor 的依赖
  StandaloneService[DI_DEPENDENCIES].forEach(([id, service]) => {
    _create(service, services);
  });
  return _create(StandaloneService, services);
}

function _create(ctor, args){
  return Reflect.constructor(StandaloneService, services;
}
```
