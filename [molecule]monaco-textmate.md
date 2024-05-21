[molecule]基于 textmate 替换 monaco 默认的 Monarch

# 为什么要做替换？
monaco 默认支持的语言太少，而目前社区丰富的都是 textmate，包括 VSCode 也是支持 textmate 的。所以用 textmate 来尝试实现语法高亮。

# How

以下用到的相关依赖版本如下：
| 依赖 | 版本 |
| --- | --- |
| `vscode-oniguruma` | `^2.0.1` |
| `vscode-textmate` | `6.x` |

## 加载 oniguruma 文件
首先，我们需要加在 [oniguruma](https://en.wikipedia.org/wiki/Oniguruma)，这里我们直接安装 `vscode-oniguruma` 库，然后参考 [VSCode](https://github.com/microsoft/vscode/blob/829230a5a83768a3494ebbc61144e7cde9105c73/src/vs/workbench/services/textMate/browser/textMateService.ts#L33-L40) 加载 `onig.wasm` 的方式加载相关文件。

```ts
// Taken from https://github.com/microsoft/vscode/blob/829230a5a83768a3494ebbc61144e7cde9105c73/src/vs/workbench/services/textMate/browser/textMateService.ts#L33-L40
private async loadVSCodeOnigurumWASM(): Promise<Response | ArrayBuffer> {
    const response = await fetch(this.onigurumPath);
    const contentType = response.headers.get('content-type');
    if (contentType === 'application/wasm') {
        return response;
    }

    // Using the response directly only works if the server sets the MIME type 'application/wasm'.
    // Otherwise, a TypeError is thrown when using the streaming compiler.
    // We therefore use the non-streaming compiler :(.
    return await response.arrayBuffer();
}
```

获取到相关的 WebAssembly 文件之后，借助 `vscode-oniguruma` 提供的 `loadWASM` 函数加载该文件。

## 注册语言的 configuration 和 grammars 文件

这里需要借助 `vscode-textmate` 的 `Registry` 类来获取相关的语法文件。

初始化内容大致如下，接下来解释每个 Option 项的作用

```ts
this.registry = new Registry({
    onigLib,

    async loadGrammar(scopeName: ScopeName): Promise<IRawGrammar | null> {
        // ...
    },

    getInjections(scopeName: ScopeName): string[] | undefined {
        // ...
    },

    // Note that nothing will display without the theme!
    theme,
});
```

### onigLib
 `onigLib` 接收类型为 `Promise<IOnigLib>;`，其中 `IOnigLib` 的类型又如下：
```ts
/**
 * A registry helper that can locate grammar file paths given scope names.
 */
export interface RegistryOptions {
    onigLib: Promise<IOnigLib>;
    theme?: IRawTheme;
    colorMap?: string[];
    loadGrammar(scopeName: string): Promise<IRawGrammar | undefined | null>;
    getInjections?(scopeName: string): string[] | undefined;
}

export interface IOnigLib {
    createOnigScanner(sources: string[]): OnigScanner;
    createOnigString(str: string): OnigString;
}
```
这里我们我们自己构造，`vscode-oniguruma` 提供了默认函数，直接引用即可。如下：
```ts
import { createOnigScanner, createOnigString } from 'vscode-oniguruma';
const onigLib = Promise.resolve({
    createOnigScanner,
    createOnigString,
});
```

### loadGrammar
loadGrammar 顾名思义是加载语法文件的方法，由上面提到的 `RegistryOptions` 类型定义中可以看到，参数是 scopeName，返回值 `Promise<IRawGrammar>` 对象。

#### scopeName

scopeName 的含义是用来标识和区分不同语法高亮规则的一个唯一标识符，那么其和 id 的区别在于，该字段同时还可以表达语法高亮的结构关系。

例如：`JavaScript` 的 scopeName 是 `source.js`(可以在 [vscode/extensions/javascript/syntaxes/JavaScript.tmLanguage.json](https://github.com/microsoft/vscode/blob/5a514567983683ec5561c1e5ea45c286601164d6/extensions/javascript/syntaxes/JavaScript.tmLanguage.json) 中看到)

再例如：`css` 的 scopeName 是 `source.css`，而 `sass` 的 scopeName 是 `source.css.scss`。这表明了 sass 语法是 css 语法的超集。

#### IRawGrammar

这里先不详细给出 `IRawGrammar` 类型的定义，如果有需要可以自行到 `vscode-textmate` 库里查看。我们只需要知道 `loadGrammar` 函数需要返回语法文件。

```ts
const scopeNameInfo = grammars[scopeName];
if (scopeNameInfo == null) {
    return null;
}

const { type, grammar } = await fetchGrammar(scopeName);
// If this is a JSON grammar, filePath must be specified with a `.json`
// file extension or else parseRawGrammar() will assume it is a PLIST
// grammar.
return parseRawGrammar(grammar, `grammar.${type}`);
```
这里的 `grammars` 是 `Record<string, any>` 的类型，主要定义了注册的语法名称等相关的语法元数据，其中需要定义该语法相关的语法文件地址。

`fetchGrammar` 函数是通过 `fetch` 函数获取语法文件，额外的点在于需要区分语法文件是 `.json` 还是 `.plist`。

```ts
private fetchGrammar = async (scopeName: string): Promise<TextMateGrammar> => {
    const { path } = this.grammars[scopeName];
    const uri = `/grammars/${path}`;
    const response = await fetch(uri);
    const grammar = await response.text();
    const type = path.endsWith('.json') ? 'json' : 'plist';
    return { type, grammar };
};
```

#### .tmLanguage.json

通常来说，语法文件为 `.tmLanguage.json` 或以 `.plist` 结尾。

##### name
name 字段表示定义的语法名称

##### scopeName
如上所述，scopeName 的含义是用来标识和区分不同语法高亮规则的一个唯一标识符。

##### patterns
`patterns` 字段定义了一组模式，用于匹配和识别文本中的特定语法结构。每个模式可以描述如何匹配特定的语法元素，如关键字、注释、字符串等。

例如：在 [`JSON.tmLanguage.json`](https://github.com/microsoft/vscode/blob/51591f83a103aec9acb2df4e64467d507107f776/extensions/json/syntaxes/JSON.tmLanguage.json#L10-L14) 文件中有这么一段代码，表示引入一段名为 `value` 的命名模式（规则）。而 `value` 具体的模式可以在 [`repository`](https://github.com/microsoft/vscode/blob/51591f83a103aec9acb2df4e64467d507107f776/extensions/json/syntaxes/JSON.tmLanguage.json#L190-L211) 字段中找到
```js
"patterns": [
    {
        "include": "#value"
    }
],
```

##### repository
`repository` 是一个对象，其中包含了一组命名模式。每个命名模式可以包含一组 patterns，这些模式将被引入到使用 include 的地方。

### getInjections

有部分语法存在插入的情况，例如在 vue 的语法中，存在 html 的语法，这里就可以通过 injections 字段来进行扩展。

```ts
/**
 * For the given scope, returns a list of additional grammars that should be
 * "injected into" it (i.e., a list of grammars that want to extend the
 * specified `scopeName`). The most common example is other grammars that
 * want to "inject themselves" into the `text.html.markdown` scope so they
 * can be used with fenced code blocks.
 *
 * In the manifest of a VS Code extension, a grammar signals that it wants
 * to do this via the "injectTo" property:
 * https://code.visualstudio.com/api/language-extensions/syntax-highlight-guide#injection-grammars
 */
getInjections(scopeName: ScopeName): string[] | undefined {
    const grammar = grammars[scopeName];
    return grammar ? grammar.injections : undefined;
},
```

### theme
这里的 theme 需要是兼容 VSCode 的 theme，并且如果不赋值，则高亮不生效。

`molecule` 在实现基于 `monaco-editor` 的 theme 转换成 `IRawTheme` 的函数主要参考了 [theia](https://github.com/eclipse-theia/theia/blob/38eb31945130bb68fc793d4291d8d9f416541cdb/packages/monaco/src/browser/textmate/monaco-theme-registry.ts#L78) 中的实现

### TokensProviderCache
在实现了以上 registry 后额外实现一个 `TokensProviderCache` 类。该类的作用是防止语法的重复 tokensProvider 的重复加载。

主要实现了一个 `createEncodedTokensProvider` 方法，该方法的返回值会提供给 `monaco.languages.setTokensProvider` 进行注册。

该方法的实现主要参考了 [monaco-tm](https://github.com/bolinfest/monaco-tm/blob/908f1ca0cab3e82823cb465108ae86ee2b4ba3fc/src/providers.ts#L155-L211)。


## 注册语言
刚才是把语言相关元数据整理并封装，接下来需要在 monaco-editor 中注册，并且懒加载相关语言。

```ts
/**
 * This function needs to be called before monaco.editor.create().
 *
 * @param languages the set of languages Monaco must know about up front.
 * @param fetchLanguageInfo fetches full language configuration on demand.
 * @param monaco instance of Monaco on which to register languages information.
 */
export function registerLanguages(
    languages: monaco.languages.ILanguageExtensionPoint[],
    fetchLanguageInfo: (language: LanguageId) => Promise<LanguageInfo>,
    monaco: Monaco
) {
    // We have to register all of the languages with Monaco synchronously before
    // we can configure them.
    for (const extensionPoint of languages) {
        // Recall that the id is a short name like 'cpp' or 'java'.
        const { id: languageId } = extensionPoint;
        monaco.languages.register(extensionPoint);

        // Lazy-load the tokens provider and configuration data.
        monaco.languages.onLanguage(languageId, async () => {
            const { tokensProvider, configuration } = await fetchLanguageInfo(languageId);

            if (tokensProvider != null) {
                monaco.languages.setTokensProvider(languageId, tokensProvider);
            }

            if (configuration != null) {
                monaco.languages.setLanguageConfiguration(languageId, configuration);
            }
        });
    }
}
```

### fetchLanguageInfo
这里的 `fetchLanguageInfo` 函数是用来获取 `tokensProvider` 和 `configuration` 的。

前者通过我们刚才提到的 `TokensProviderCache` 类获取，后者直接通过 `window.fetch` 来获取配置文件即可。

需要注意一下，在获取到 `configuration` 的之后，需要做一个 hydrate。

```ts
private fetchConfiguration = async (language: string): Promise<monaco.languages.LanguageConfiguration> => {
    const uri = `/configurations/${language}.json`;
    const response = await fetch(uri);
    const rawConfiguration = await response.text();
    return rehydrateRegexps(rawConfiguration);
};

/**
 * Configuration data is read from JSON and JSONC files, which cannot contain
 * regular expression literals. Although Monarch grammars will often accept
 * either the source of a RegExp as a string OR a RegExp object, certain Monaco
 * APIs accept only a RegExp object, so we must "rehydrate" those, as appropriate.
 *
 * It would probably save everyone a lot of trouble if we updated the APIs to
 * accept a RegExp or a string literal. Possibly a small struct if flags need
 * to be specified to the RegExp constructor.
 */
export function rehydrateRegexps(rawConfiguration: string): monaco.languages.LanguageConfiguration {
  const out = JSON.parse(rawConfiguration);
  for (const property of REGEXP_PROPERTIES) {
    const value = getProp(out, property);
    if (typeof value === 'string') {
      setProp(out, property, new RegExp(value));
    }
  }
  return out;
}
```

#### configuration
该文件通常以 `.json` 的格式存在，定义了语言的行为、文件关联、注释样式等。

## 主题生效
在主题生效之后，需要进行一个 `injectCSS` 的操作。直接参考 monaco-tm 中的实现。

```ts
/**
 * Be sure this is done after Monaco injects its default styles so that the
 * injected CSS overrides the defaults.
 */
injectCSS() {
    const cssColors = this.registry.getColorMap();
    const colorMap = cssColors.map(Color.Format.CSS.parseHex);
    // This is needed to ensure the minimap gets the right colors.
    TokenizationRegistry.setColorMap(colorMap);
    const css = generateTokensCSSForColorMap(colorMap);
    const style = createStyleElementForColorsCSS();

    style.innerHTML = css;
}
```

需要注意，切换主题之后需要更新 `Register` 中的 theme，然后再度调用 injectCSS 更新样式

# 参考
大量参考了其他开源项目或博客。

- [1].[WebIDE 的开发记录其五（monaco-editor + textmate）](https://ubug.io/blog/workpad-part-5)
- [2].[monaco-tm](https://github.com/bolinfest/monaco-tm)
- [3].[手把手教你实现在Monaco Editor中使用VSCode主题](https://www.cnblogs.com/wanglinmantan/p/15345204.html)
- [4].[theia](https://github.com/eclipse-theia/theia)