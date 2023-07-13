基于知乎的 [vscode是如何做到动态切换主题的](https://www.zhihu.com/question/524745185/answer/2412775919) 这篇回答，继续深入探究 vscode 的相关实现原理。

首先，先找到 `src/vs/workbench/browser/web.factory.ts` 文件，该文件中有一个 create 函数，其类型如下：
```ts
/**

* Creates the workbench with the provided options in the provided container.

*

* @param domElement the container to create the workbench in

* @param options for setting up the workbench

*/
export function create(domElement: HTMLElement, options: IWorkbenchConstructionOptions): IDisposable {}
```

该函数的主要作用是用提供的 `options` 在提供的 `container` 中创建 `workbench`。

其函数体中存在如下代码：
```ts
export function create(domElement: HTMLElement, options: IWorkbenchConstructionOptions): IDisposable {
	// ...
	// Startup workbench and resolve waiters
	let instantiatedWorkbench: IWorkbench | undefined = undefined;
	new BrowserMain(domElement, options).open().then(workbench => {
		instantiatedWorkbench = workbench;	
		workbenchPromise.complete(workbench);
	});
	// ...
}
```

通过 `BrowserMain` 类构造界面，再调用 `open` 方法返回 `workbench`.

先看 `BrowserMain` 类的构造函数，其代码位于 `src/vs/workbench/browser/web.main.ts`
```js src/vs/workbench/browser/web.main.ts
class BrowserMain extends Disposable {
	constructor(
		private readonly domElement: HTMLElement,
		private readonly configuration: IWorkbenchConstructionOptions
	) {
		super();
		this.init();
	}
	
	private init(): void {
		// Browser config
		setFullscreen(!!detectFullscreen());
	}
}
```

`init` 方法主要是负责屏幕的全屏相关设置，与探究 colorTheme 并无关，故跳过。

再看 `open` 方法
```js
class BrowserMain extends Disposable {
	async open(): Promise<IWorkbench> {
		// Init services and wait for DOM to be ready in parallel
		const [services] = await Promise.all([this.initServices(), domContentLoaded()]);
		// Create Workbench
		const workbench = new Workbench(this.domElement, undefined, services.serviceCollection, services.logService);
		// Listeners
		this.registerListeners(workbench);
		// Startup
		const instantiationService = workbench.startup();
		// ...
	}
}

```
可以看到，这里初始化了所有的 `services`，然后通过初始化 `Workbench` 类，并调用其 startup 方法获得 `instantiationService`

到这里，整理线索可得
`create function -> BrowserMain open -> Workbench constructor -> Workbench startup`

`Workbench` 类位于 `src/vs/workbench/browser/workbench.ts`, 其构造器如下：
```js
export class Workbench extends Layout {
	constructor(
		parent: HTMLElement,
		private readonly options: IWorkbenchOptions | undefined,
		private readonly serviceCollection: ServiceCollection,
		logService: ILogService
	) {
		super(parent);
		// Perf: measure workbench startup time
		mark('code/willStartWorkbench');
		this.registerErrorHandler(logService);
	}
}
```

可以看到构造器初始化做了 3 件事，分别是「调用父类」、「打标记」（主要为了性能优化的衡量）、「注册错误事件句柄」。
后两者和 colorTheme 无关，在 `Layout` 的构造器中也发现与 colorTheme 无关，故发现 `Workbench` 的初始化与 colorTheme 无关，再看 `startup` 方法

```js
startup(): IInstantiationService {
	// ...
	// Render Workbench
	this.renderWorkbench(instantiationService, accessor.get(INotificationService) as NotificationService, storageService, configurationService);
	// ...
}
```

其中，存在的 `renderWorkbench` 主要负责对工作台做渲染，该方法中通过创建 `Parts` 来渲染各个部分

<details>
    <summary>Parts</summary>
    Vscode 把各个区域划分为 Parts
</details>

同时，在当前文件的同级目录下 `src/vs/workbench/browser/parts` 存在 `parts` 目录，主要存放 Parts 相关类。

打开任意一个 Part ，都可以看到其是继承了 Part 类的

```js
export class StatusbarPart extends Part {}
export class SidebarPart extends CompositePart<PaneComposite> {}
export abstract class CompositePart<T extends Composite> extends Part {}
export class ActivitybarPart extends Part {}
```

不论是直接继承还是间接继承，最终都会来到 Part 类，而 Part 类是来源于 Themable 类的，该类上存在 `updateStyles` 方法，是为了让子类去重写的。

整理线索可得
`Workbench startup -> Workbenc renderWorkbench -> Parts -> Themable -> Themable updateStyles` 


在大部分的 `updateStyles` 方法中，都存在 `this.getColor` 的相关代码，相关实现在 `Themable` 类上
```js
export class Themable extends Disposable {
	protected getColor(id: string, modify?: (color: Color, theme: IColorTheme) => Color): string | null {
		let color = this.theme.getColor(id);

		if (color && modify) {
			color = modify(color, this.theme);
		}

		return color ? color.toString() : null;
	}
}
```

该方法接收两个参数，主要需要一个 string 类型的 id 参数，会通过 id 参数到 `themeService` 中获取 `color` 属性

而 `themeService` 是通过依赖注入的方式获取到的值，
