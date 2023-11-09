# Molecule 指路

## What Molecule

指路：[README](https://github.com/DTStack/molecule)

## Why Molecule

在开始之前，我们需要先理解 Molecule 的定位是什么？

- lightweight
- Web IDE
- UI Framework

### lightweight

何为轻量级？这里需要结合 Web IDE 来说。

首先，社区中不少做 Web IDE 的框架。例如 [theia](https://theia-ide.org/) 或者 [opensumi](https://opensumi.com/zh)。

上述框架在实现 Web IDE 的场景中确实非常优秀，有完整的 Backend 通信机制，Terminal 相关实现，以及各种插件机制。

但是在某些场景下，过多引入不必要的实现会导致当前应用过于臃肿。

例如：在某些情况下，我们只想引入简单的 markdown 编辑器，同时针对 markdown 文件做一些编排，这时候引入 theia 或者 opensumi 就显得过于臃肿。

轻量级的 Molecule 意味着我们可以在不引入过多依赖的情况下，快速的实现一个 Web IDE。

### Web IDE

Molecule 作为一个 Web IDE，具备了 IDE 的基础功能。包括

- 支持 Workbench 模块，其下分为多种子模块，包括 IDE 常见的 FileSystem，Menus 等
- 支持 ColorTheme 的主题切换
- 支持国际化
- 支持快捷键，包括 Command Palette
- 支持 QuickAccess Panel
- 支持 extensions 扩展和实现用户功能

### UI Framework

Molecule 同时也做为一个 UI 框架，并非一整套完整的 Web IDE 解决方案。这一点也是 Molecule 相对于 theia 或者 opensumi 来说更加轻量级的点。Molecule 抛弃了一些不必要的实现，例如 Terminal，Debug 等。

同时也解耦了服务端，所有交互都交由用户去定义。

Molecule 只是提供了一些基础的 UI 组件，例如 Tabs，Menus，Dialog 等。并将其组合成 Workbench。

同时，基于 Workbench 之上，我们整合了 Command Palette 和 QuickAccess 等模块。

## How Molecule

Molecule 在数据处理层面使用了 MVC 架构。

- 通过 Model 层来管理数据
- Service 层来初始化数据，且在 Service 中定义了各种数据的原子化操作
- Controller 接受交互行为的事情输入，并将其转化为 Service 中的原子化操作

### 如何实现 MVC

Model 层非常容易实现，我们只需要定义一个 Model 类，然后在其中定义一些数据即可。

```ts
class Model {
  constuctor(private a: string) {}
}
```

Service 层也非常容易实现，我们只需要定义一个 Service 类，然后在其中定义一些数据初始化的方法，以及一些原子化操作即可。

```ts
class Service {
  constuctor() {
    this.state = new Model();
  }

  public handleA = () => {
    //...
  };
}
```

controller 也以此类推。

```ts
class Controller {
  constuctor() {
    this.service = new Service();
  }

  public handleA = () => {
    this.service.handleA();
  };
}
```

这里会有如下难点：

1. 如何确保不同的 Service 中修改同一份数据
2. 如何让 View 层的交互行为能够触发 Controller 中的方法，并且 Service 中的数据修改可以出发 View 层的更新

#### 如何确保不同的 Service 中修改同一份数据

这里用到了借助 tsyringe 以及 IOC 实现**应用级别的单例模式**。

即每一个 Molecule 应用初始化一个实例，在该实例中，我们初始化 ServiceContainer，将全部的 Service 都初始化到该容器中。实现容器内的单例。

然后借助 IOC 的特性，我们在不同的 Controller 都注入容器中的单例。

> 这里 Model 并不需要是单例的，因为考虑到 Model 层是非常简单的数据结构声明，没有必要单例。

#### 如何解决 View 层和 Controller&Serice 的交互

由于 View 层我们考虑使用 React 组件实现，所以我们需要将 React 组件和 Controller&Service 进行绑定。

首先，我们实现 React 和 Service 的绑定。即实现一个 HOC 或者 hook 来确保 React 组件能够获取到 Service 中的数据。

同时，Service 需要实现一个方法，用于通知 React 组件更新的方法。这样，当 Service 中的数据发生变化时，我们可以通过该方法来触发 React 组件的更新。

```ts
class EventEmitter {
  setState = () => {
    this.emit("onUpdateState");
  };
  onUpdateState = () => {
    this.subscribe("onUpdateState");
  };
}

class Service extends EventEmitter {
  constuctor() {
    this.state = new Model();
  }

  public handleA = () => {
    //...
    this.setState(xxx);
  };
}
```

如上，借助事件订阅机制，并且我们确保 Service 所有方法的更新都调用 setState 的方法。那么在 View 中，我们只需要监听 onUpdateState 事件即可。

```ts
function useConnect(serviceName: string) {
  const [_, setData] = useState(false);
  const forceUpdate = setData((d) => !d);
  const data = useRef();

  useEffect(() => {
    const service = container.get(serviceName);
    service.onUpdateState(() => {
      forceUpdate();
    });
  }, []);

  return data.current;
}

function View() {
  const service = useConnect();
  return <div>{service.a}</div>;
}
```

如上，实现了一个 useConnect 的 hook，监听了 onUpdateState 事件，并且在事件触发时，强制更新 React 组件。_（Class 组件实现 HOC 即可）_

但是如上实现会有一个问题，即当前组件只用到了 a 字段，但是 onUpdateState 事件触发时，我们强制更新了整个组件，这样会导致组件的重新渲染，即使组件中的其他字段并没有发生变化。

这里我们需要针对 useConnect 做性能优化。即获取 deps，并配合 Object.is 判断是否数据变更。

详细代码参考 [useConnector](https://github.com/DTStack/molecule/blob/bd81f9b900de62a61b4e4d5ee9ab63f979bf6c2d/src/client/hooks/useConnector.ts)

通过类似的思路，也可以实现 View 和 Controller 的绑定，在这里不再赘述。

### 实现 Service 的原子化操作

实现 Service 的原子化操作会简单一些，就是对数据的一些增删改查。

略微复杂一些的是针对 FileSystem 的树结构的增删改查。需要配合某些 WeakMap 对树结构进行缓存。

防止每次增删改查的时候都需要对树进行遍历。是一个用空间换时间的思路。

### 如何实现 Command Palette

在实现了 MVC 架构后，我们需要实现 Command Palette。即思考如何让用户用最简单的方式注册快捷键。

首先，我们考虑 VSCode 的插件实现方式。VSCode 的插件实现方式是通过 package.json 中的 contributes.commands 字段来实现的。

而 Molecule 则通过在插件中定义 contributes 实现:

```ts
contributes: {
        [IContributeType.Commands]: [
            QuickSelectThemeAction,
        ],
    },
```

这一块设计到 extensions 的设计思路。详见 [extensions](#如何实现-extensions)。

而 Command Palette 的实现难点在于**如何将 monaco-editor 的 actions 触发从 monaco-editor 中抽离**出来，以便于我们可以在 monaco-editor 之外的地方注册快捷键。

首先，我们通过既定的 _（这个格式和 VSCode 注册快捷键的格式一致）_ 格式，将 actions 注册到 monaco-editor 的 KeybindingsRegistry 中。

如此之后，我们即将快捷键注册到了 monaco-editor 之中。但是目前只有 monaco-editor 中可以触发 actions。

接下来，我们将 override monaco-editor 中的 SimpleLayoutService 服务。这里代码比较长，详见 [molecule/monaco.ts](https://github.com/DTStack/molecule/blob/bd81f9b900de62a61b4e4d5ee9ab63f979bf6c2d/src/services/monaco.ts)。

<a href="#why-override"></a>

<details>
<summary>Why override？</summary>
<p>

这里需要理解，monaco-editor 的实现思路是基于 services 的。其通过把各种服务进行抽象和解耦，将不同的 services 进行组合，从而实现一个完整的 monaco-editor。

例如：keybinding，colorTheme，contextMenu，command 等。可以理解为其底层有一个最简单的 EditorService，该 Service 通过耦合 ContextMenu 的功能，则实现了一个具备右键菜单的 Editor。再继续耦合 ColorThemeService，则实现了具有主题切换功能的 Editor。

那么如此依赖，我们只需要找到相关的 Service，并重写相关逻辑即可。

</p>
</details>

Molecule 的做法是重写 LayoutService。_（Molecule 不止重写了 Layout，这里只涉及了 Layout，所以不提其他的）_。将 LayoutService 中的 Container 字段修改为当前 Molecule 应用的 Container。那么就实现了在整个应用中触发 actions 的事件了。

### 如何实现 QuickAccess

原理其实在 [Why Override](#why-molecule) 已经提到了。我们重写完 LayoutService 后，再重写 QuickInputService 相关内容即可。

### 如何实现 ColorTheme

这里主要是参考 VSCode 中相关实现。首先我们需要实现内置的 3 种 colorTheme，包括 dark、light、high contrast。并通过内置的 extensions 加载到 Molecule 中。

这里实现的难点如下：

- 如何实现 Color 之间的继承关系
- 如何实现主题丝滑切换，而不需要刷新当前页面

#### 如何实现 Color 之间的继承关系

需要表达继承关系，那就无法保存具体的色值。这里我们需要定义一种 Token，用于表达继承关系。

这里，我们参考了 VSCode 中的实现。通过定义 ColorToken，_（这里的 Token 来源全部参考 VSCode 的 [theme-color](https://code.visualstudio.com/api/references/theme-color)）_，每一个 ColorToken 在内存中以对象的形式存储，该对象中包含了继承关系，dark、light、high contrast 下的色号等信息。

同时，我们定义了一个 ColorThemeService，该 Service 中包含了当前 ColorTheme 的主题是什么。

详情参考 [theme](https://github.com/DTStack/molecule/blob/bd81f9b900de62a61b4e4d5ee9ab63f979bf6c2d/src/const/theme.ts)

#### 如何实现主题丝滑切换，而不需要刷新当前页面

当应用主题时，Molecule 通过解析 ColorToken，获取继承关系，生成最终的对应 Theme 下的色值。然后通过 CSS Variables 加载到 `:root` 下。

同时，Workbench 中的所有组件都需要配合使用 CSS Variables 实现。

### 如何实现 extensions

借助上面实现的 MVC 结构，我们实现了基本的 Workbench 后，我们需要将相关交互通过某种机制暴露给用户。

前文性能优化相关的部分也提到了事件订阅的机制，这里我们也借助事件订阅的机制，将交互暴露给用户。

> Controller 接受交互行为的事情输入，并将其转化为 Service 中的原子化操作

事实上，Molecule 并不会在 Controller 层直接调用 Service 中的操作。因为这会导致用户的行为无法参与到交互中。

Molecule 在 Controller 里是通过事件订阅的方式，将交互行为暴露出去。而用户根据实际情况去订阅交互行为，然后调用 Service 中的操作，从而影响数据渲染。

而 extensions 在这里就作为了用户写订阅逻辑的载体。事实上，extensions 是一个纯函数。Molecule 在应用加载初期会加载所有 extensions 包括用户提供的以及内置的。

Molecule 在加载 extensions 后，会执行所有订阅事件。也就实现了用户订阅监听到了交互行为。

### 如何实现国际化

理想的国际化实现方式是，用户只需要提供一个翻译文件，然后 Molecule 会自动将翻译文件加载到应用中。

这里的难点在于：

- 如何做到无刷新加载翻译文件
- 如何实现 monaco-editor 的国际化

#### 如何做到无刷新加载翻译文件

期望做到无刷新加载翻译文件，则需要提供 `formatMessage` 的函数，在 Workbench 的各个组件中渲染文案时，通过调用该函数来获取翻译后的文案。在切换文案后，实际上只是切换了 localeSerivce 中 active 的 locale，然后重新调用 `formatMessage` 函数即可。

这里的难点实际上并不是文件本身，而在于加载顺序的问题。

#### 如何实现 monaco-editor 的国际化

monaco-editor 的国际化实际上可以实现，但是目前调研的方案还无法在 runtime 的时候实现国际化。

Molecule 目前这一块也做得不是很好。

### 加载顺序

前文提到的加载顺序，这里着重讲一下加载顺序的问题。

Molecule 作为一个比较大的应用，其中需要加载各种 Services，Service 可能还会继续加载其他的部分。为了更好地维护项目，同时也为了项目的稳定性。我们需要保证加载顺序的正确性。

这里我们定义了一些生命周期，在指定的生命周期去做指定的事情。同时通过配置的方式将生命周期暴露给用户。

> 这里不能通过 extensions 将生命周期暴露给用户。因为当执行到某些生命周期的时候，extensions 不一定加载到应用中。

同时，我们还需要确保各个 Services 加载的顺序。如：localeSerivce 需要提前加载，extensionService 需要最后加载等。

除了上述提到的难点以外，还有部分未提及的。例如：

- 如何解决事件监听机制无法 abort 的问题
  - bailHook
- 如何解决 Workbench 中想要自定义组件的问题
  - swizzle
- 如何更好地设计 Workbench 的结构，让数据的流动更加容易维护
  - context
  - BEM
- 如何实现 BEM 需要重复生命 ClassName 的问题
  - SASS-in-JS
- 如何设计交互事件
  - 切忌过度设计
  - 参考 ddd
