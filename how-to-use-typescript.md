# 使用编译 API

> 声明：目前还不是一个稳定版本的 API - 我们(指 mircosoft )发布其为 0.5 版，并且随着时间的流逝会有所不同。第一次迭代时，会有一些瑕疵。我们鼓励社区提供反馈，以改进 API。为了允许用户在将来的版本之间转换，我们将记录每个新版本的所有 [API 重大更改](https://github.com/microsoft/TypeScript/wiki/API-Breaking-Changes)。

## 开始

首先，你需要先从 `npm` 下载安装 TypeScript >=1.6.

完成后，你需要从项目所在的任何地方链接它。如果你不是从 Node 项目中进行链接，则它将进行全局链接.

```bash
npm install -g typescript
npm link typescript
```

除此之外，你还需要一些类型定义文件(node definition file)。要获取定义文件，请运行：

```bash
npm install @types/node
```

就这些，现在，你可以尝试以下一些示例。

编译 API 具有几个主要组件：
- `Program` 是整个应用程序的 TypeScript 术语
- `CompilerHost` 代表了用户系统，拥有读取文件，检查目录，大小写敏感等的 API
- 许多 `SourceFile` 代表应用程序中的每个源文件，同时有文本和TypeScript AST

## 一个最小的编译器

此示例是一个大致的编译器，它获取 TypeScript 文件列表并将其编译为相应的 JavaScript.

我们需要通过 `createProgram` 创建一个 `Program` - 这将创建一个默认的 `CompilerHost` 以使用文件系统来获取文件。

```typescript
import * as ts from "typescript";

function compile(fileNames: string[], options: ts.CompilerOptions): void {
  let program = ts.createProgram(fileNames, options);
  let emitResult = program.emit();

  let allDiagnostics = ts
    .getPreEmitDiagnostics(program)
    .concat(emitResult.diagnostics);

  allDiagnostics.forEach(diagnostic => {
    if (diagnostic.file) {
      let { line, character } = ts.getLineAndCharacterOfPosition(diagnostic.file, diagnostic.start!);
      let message = ts.flattenDiagnosticMessageText(diagnostic.messageText, "\n");
      console.log(`${diagnostic.file.fileName} (${line + 1},${character + 1}): ${message}`);
    } else {
      console.log(ts.flattenDiagnosticMessageText(diagnostic.messageText, "\n"));
    }
  });

  let exitCode = emitResult.emitSkipped ? 1 : 0;
  console.log(`Process exiting with code '${exitCode}'.`);
  process.exit(exitCode);
}

compile(process.argv.slice(2), {
  noEmitOnError: true,
  noImplicitAny: true,
  target: ts.ScriptTarget.ES5,
  module: ts.ModuleKind.CommonJS
});
```

## 一个简单的转化函数

创建编译器不需要太多的代码行，但是您可能只想在给定 TypeScript 源的情况下获取相应的 JavaScript 输出。为此，您可以使用 `ts.transpileModule` 获取 string => string 转换仅使用两行代码。

```ts
import * as ts from "typescript";

const source = "let x: string  = 'string'";

let result = ts.transpileModule(source, { compilerOptions: { module: ts.ModuleKind.CommonJS }});

console.log(JSON.stringify(result, null 2));
```

## 从 JavaScript 文件获取 DTS

> 注：DTS 即 d.ts 文件

这仅在 TypeScript 3.7 及更高版本中有效。本示例说明如何获取 JavaScript 文件列表，并在终端中显示其生成的 d.ts 文件。

```ts
import * as ts from "typescript";

function compile(fileNames: string[], options: ts.CompilerOptions): void {
  // 创建一个带有内存 emit 的程序
  const createdFiles = {}
  const host = ts.createCompilerHost(options);
  host.writeFile = (fileName: string, contents: string) => createdFiles[fileName] = contents
  
  // 准备并 emit d.ts 文件
  const program = ts.createProgram(fileNames, options, host);
  program.emit();

  // 遍历全部输入文件
  fileNames.forEach(file => {
    console.log("### JavaScript\n")
    console.log(host.readFile(file))

    console.log("### Type Definition\n")
    const dts = file.replace(".js", ".d.ts")
    console.log(createdFiles[dts])
  })
}

// 运行编译器
compile(process.argv.slice(2), {
  allowJs: true,
  declaration: true,
  emitDeclarationOnly: true,
});
```

## 重印部分 TypeScript 文件

本示例将记录 TypeScript 或 JavaScript 源文件的小节，当您希望应用程序的代码是 SSOT 时，此模式非常有用。例如，通过 JSDoc 注释展示导出。

```ts
import * as ts from "typescript";

/**
 * 从源文件中打印指定节点
 * 
 * @param file 文件路径
 * @param identifiers 可用的顶级标识符
 */
function extract(file: string, identifiers: string[]): void {
  // 创建一个程序，然后获取源文件以分析其 AST
  let program = ts.createProgram([file], { allowJs: true });
  const sourceFile = program.getSourceFile(file);
  
  // 为了打印 AST，我们将会使用 TypeScript 的打印机
  const printer = ts.createPrinter({ newLine: ts.NewLineKind.LineFeed });

  // 为了给出建设性的错误消息，需要跟踪找到和未找到的标识符
  const unfoundNodes = [], foundNodes = [];

  // 遍历 AST 的根结点
  ts.forEachChild(sourceFile, node => {
    let name = "";
    
    // 这是一组不完整的 AST 节点，可能具有顶级标识符
    // 这留给您展开此列表，您可以使用
    // https://ts-ast-viewer.com/ 查看文件的AST，然后使用相同的模式
    // 如下
    if (ts.isFunctionDeclaration(node)) {
      name = node.name.text;
      // 打印时隐藏方法主体
      node.body = undefined;
    } else if (ts.isVariableStatement(node)) {
      name = node.declarationList.declarations[0].name.getText(sourceFile);
    } else if (ts.isInterfaceDeclaration(node)){
      name = node.name.text
    }

    const container = identifiers.includes(name) ? foundNodes : unfoundNodes;
    container.push([name, node]);
  });

  // 既不是打印找到的节点，也不能提供找到的标识符的列表
  if (!foundNodes.length) {
    console.log(`Could not find any of ${identifiers.join(", ")} in ${file}, found: ${unfoundNodes.filter(f => f[0]).map(f => f[0]).join(", ")}.`);
    process.exitCode = 1;
  } else {
    foundNodes.map(f => {
      const [name, node] = f;
      console.log("### " + name + "\n");
      console.log(printer.printNode(ts.EmitHint.Unspecified, node, sourceFile)) + "\n";
    });
  }
}

// 使用脚本的参数运行 extract 函数
extract(process.argv[2], process.argv.slice(3));
```

## 遍历 AST 实现一个小型的 linter 工具

`Node` 是 TypeScript AST 的根接口。通常，我们使用 `forEachChild` 函数以递归方式来遍历树。这是一种访问者模式，通常来说可以提供更大的灵活性。

作为一个如何遍历文件的 AST 示例，考虑一个最小化的 linter 工具实现以下操作：
- 检查所有循环的构造体是否用花括号括起来。
- 检查所有 if/else 是否用花括号括起来。
- 使用 "stricter" 的等号运算符（`===`/`!==`）代替 "loose" 的等号运算符（`==`/`!=`）。

```ts
import { readFileSync } from "fs";
import * as ts from "typescript";

export function delint(sourceFile: ts.SourceFile) {
  delintNode(sourceFile);

  function delintNode(node: ts.Node) {
    switch (node.kind) {
      case ts.SyntaxKind.ForStatement:
      case ts.SyntaxKind.ForInStatement:
      case ts.SyntaxKind.WhileStatement:
      case ts.SyntaxKind.DoStatement:
        if ((node as ts.IterationStatement).statement.kind !== ts.SyntaxKind.Block) {
          report(
            node,
            'A looping statement\'s contents should be wrapped in a block body.'
          );
        }
        break;

      case ts.SyntaxKind.IfStatement:
        const ifStatement = node as ts.IfStatement;
        if (ifStatement.thenStatement.kind !== ts.SyntaxKind.Block) {
          report(ifStatement.thenStatement, 'An if statement\'s contents should be wrapped in a block body.');
        }
        if (
          ifStatement.elseStatement &&
          ifStatement.elseStatement.kind !== ts.SyntaxKind.Block &&
          ifStatement.elseStatement.kind !== ts.SyntaxKind.IfStatement
        ) {
          report(
            ifStatement.elseStatement,
            'An else statement\'s contents should be wrapped in a block body.'
          );
        }
        break;

      case ts.SyntaxKind.BinaryExpression:
        const op = (node as ts.BinaryExpression).operatorToken.kind;
        if (op === ts.SyntaxKind.EqualsEqualsToken || op === ts.SyntaxKind.ExclamationEqualsToken) {
          report(node, 'Use \'===\' and \'!==\'.');
        }
        break;
    }

    ts.forEachChild(node, delintNode);
  }

  function report(node: ts.Node, message: string) {
    const { line, character } = sourceFile.getLineAndCharacterOfPosition(node.getStart());
    console.log(`${sourceFile.fileName} (${line + 1},${character + 1}): ${message}`);
  }
}

const fileNames = process.argv.slice(2);
fileNames.forEach(fileName => {
  // 解析文件
  const sourceFile = ts.createSourceFile(
    fileName,
    readFileSync(fileName).toString(),
    ts.ScriptTarget.ES2015,
    /*setParentNodes */ true
  );

  // delint it
  delint(sourceFile);
});
```

在此示例中，我们不需要创建类型检查器（type checker），因为我们要做的只是遍历每个 `SourceFile`。

所有 `ts.SyntaxKind` 的类型都可以在[这里](https://github.com/microsoft/TypeScript/blob/7c14aff09383f3814d7aae1406b5b2707b72b479/lib/typescript.d.ts#L78)找到。

## 编写一个增量程序监听器

TypeScript 2.7 引入了两个新的 API：一个用于创建 "watcher" 的程序，该程序提供一组 API 来触发重建，以及 "watcher" 可以利用的"builder" API. `BuilderProgram` 是一个 `Program` 实例，可以智能地缓存错误并在以前的编译模块上触发，无论该模块的依赖关系是否进行级联更新的情况。watcher 可以借助构建器程序实例来实现仅更新编译中受影响文件的结果（例如发现错误并触发）。这可以加速包含许多文件的大型项目。

该 API 在编译器内部被使用去实现其 `--watch` 模式，但也可以被其他工具利用，如下所示：

```ts
import ts = require("typescript");

const formatHost: ts.FormatDiagnosticsHost = {
  getCanonicalFileName: path => path,
  getCurrentDirectory: ts.sys.getCurrentDirectory,
  getNewLine: () => ts.sys.newLine
};

function watchMain() {
  const configPath = ts.findConfigFile(
    /*searchPath*/ "./",
    ts.sys.fileExists,
    "tsconfig.json"
  );
  if (!configPath) {
    throw new Error("Could not find a valid 'tsconfig.json'.");
  }

  // TypeScript 可以使用多种不同的程序创建“策略”：
  //  * ts.createEmitAndSemanticDiagnosticsBuilderProgram,
  //  * ts.createSemanticDiagnosticsBuilderProgram
  //  * ts.createAbstractBuilder
  // 前两种产生“生成器程序”。他们使用增量策略
  // 去重新检查和触发其内容可能已更改的文件
  // 或者其依赖项可能会发生更改，从而可能影响先前的类型检查和触发更改
  // 最后一个使用普通程序，该程序在每次更改后都会进行完整类型检查
  //  `createEmitAndSemanticDiagnosticsBuilderProgram` 和 `createSemanticDiagnosticsBuilderProgram` 的唯一区别在于是否触发
  // 对于纯粹的类型检查方案，或者当另一个工具/进程句柄触发时，
  // 使用 `createSemanticDiagnosticsBuilderProgram` 可能更可取
  const createProgram = ts.createSemanticDiagnosticsBuilderProgram;

  // 注意，`createWatchCompilerHost` 还有另一个重载，它需要一些根文件。
  const host = ts.createWatchCompilerHost(
    configPath,
    {},
    ts.sys,
    createProgram,
    reportDiagnostic,
    reportWatchStatusChanged
  );

  // 从技术上讲，您可以覆盖主机上的任何给定钩子，尽管您可能不需要
  // 注意，我们认为 `origCreateProgram` 和 `origPostProgramCreate` 根本不使用 `this`。
  const origCreateProgram = host.createProgram;
  host.createProgram = (rootNames: ReadonlyArray<string>, options, host, oldProgram) => {
    console.log("** We're about to create the program! **");
    return origCreateProgram(rootNames, options, host, oldProgram);
  };
  const origPostProgramCreate = host.afterProgramCreate;

  host.afterProgramCreate = program => {
    console.log("** We finished making the program! **");
    origPostProgramCreate!(program);
  };

  // `createWatchProgram` 创建一个初始程序，监视文件，并随时间更新该程序。
  ts.createWatchProgram(host);
}

function reportDiagnostic(diagnostic: ts.Diagnostic) {
  console.error("Error", diagnostic.code, ":", ts.flattenDiagnosticMessageText( diagnostic.messageText, formatHost.getNewLine()));
}

/**
 * 每次 watch 状态改变时，都会打印诊断信息
 * 这主要用于诸如 “开始编译” 或 “编译完成” 之类的消息。
 */
function reportWatchStatusChanged(diagnostic: ts.Diagnostic) {
  console.info(ts.formatDiagnostic(diagnostic, formatHost));
}

watchMain();
```

## 增量构建支持语言服务

> 有关更多详细信息，请参阅 [Using the Language Service API](https://github.com/microsoft/TypeScript/wiki/Using-the-Language-Service-API) 页面。

服务层提供了一组附加实用程序，可以帮助简化一些复杂的场景。在下面的代码段中，我们将尝试构建一个增量构建服务器，该服务器监听一组文件并仅更新已更改文件的输出。我们将通过创建 `LanguageService` 对象来实现这一点。与上一个示例中的程序类似，我们需要一个 `LanguageServiceHost`。`LanguageServiceHost` 通过 `version`，`isOpen` 标志和 `ScriptSnapshot` 增强了文件的概念。`version` 允许语言服务跟踪文件的更改。 `isOpen` 告诉语言服务在文件使用过程中将 AST 保留在内存中。`ScriptSnapshot` 是文本的抽象，允许语言服务查询更改。

如果您只是尝试实现类监听器风格的功能，我们建议您探索上面的监听器 API。

```ts
import * as fs from "fs";
import * as ts from "typescript";

function watch(rootFileNames: string[], options: ts.CompilerOptions) {
  const files: ts.MapLike<{ version: number }> = {};

  // 初始化文件列表
  rootFileNames.forEach(fileName => {
    files[fileName] = { version: 0 };
  });

  // 创建语言服务主机以允许 LS（language service） 与主机进行通信
  const servicesHost: ts.LanguageServiceHost = {
    getScriptFileNames: () => rootFileNames,
    getScriptVersion: fileName =>
      files[fileName] && files[fileName].version.toString(),
    getScriptSnapshot: fileName => {
      if (!fs.existsSync(fileName)) {
        return undefined;
      }

      return ts.ScriptSnapshot.fromString(fs.readFileSync(fileName).toString());
    },
    getCurrentDirectory: () => process.cwd(),
    getCompilationSettings: () => options,
    getDefaultLibFileName: options => ts.getDefaultLibFilePath(options),
    fileExists: ts.sys.fileExists,
    readFile: ts.sys.readFile,
    readDirectory: ts.sys.readDirectory,
    directoryExists: ts.sys.directoryExists,
    getDirectories: ts.sys.getDirectories,
  };

  // 创建语言服务文件
  const services = ts.createLanguageService(servicesHost, ts.createDocumentRegistry());

  // 开始监听文件
  rootFileNames.forEach(fileName => {
    // 第一次触发全部文件
    emitFile(fileName);

    // 添加对文件的监听以处理下一次文件变更
    fs.watchFile(fileName, { persistent: true, interval: 250 }, (curr, prev) => {
      // 检查时间戳
      if (+curr.mtime <= +prev.mtime) {
        return;
      }

      // 更新版本以表明文件已更改
      files[fileName].version++;

      // 将文件变更写入磁盘
      emitFile(fileName);
    });
  });

  function emitFile(fileName: string) {
    let output = services.getEmitOutput(fileName);

    if (!output.emitSkipped) {
      console.log(`Emitting ${fileName}`);
    } else {
      console.log(`Emitting ${fileName} failed`);
      logErrors(fileName);
    }

    output.outputFiles.forEach(o => {
      fs.writeFileSync(o.name, o.text, "utf8");
    });
  }

  function logErrors(fileName: string) {
    let allDiagnostics = services
      .getCompilerOptionsDiagnostics()
      .concat(services.getSyntacticDiagnostics(fileName))
      .concat(services.getSemanticDiagnostics(fileName));

    allDiagnostics.forEach(diagnostic => {
      let message = ts.flattenDiagnosticMessageText(diagnostic.messageText, "\n");
      if (diagnostic.file) {
        let { line, character } = diagnostic.file.getLineAndCharacterOfPosition(
          diagnostic.start!
        );
        console.log(`  Error ${diagnostic.file.fileName} (${line + 1},${character +1}): ${message}`);
      } else {
        console.log(`  Error: ${message}`);
      }
    });
  }
}

// 初始化当前目录中的全部 .ts 文件作为程序所需要的文件
const currentDirectoryFiles = fs
  .readdirSync(process.cwd())
  .filter(fileName => fileName.length >= 3 && fileName.substr(fileName.length - 3, 3) === ".ts");

// 开启监听器
watch(currentDirectoryFiles, { module: ts.ModuleKind.CommonJS });
```

## 自定义模块解析

您可以通过实现可选方法来重载编译器解析模块的标准方法：`CompilerHost.resolveModuleNames`:

> CompilerHost.resolveModuleNames(moduleNames: string[], containingFile: string): string[]

该方法通过获取文件中的模块名列表，则会返回一个长度为 `moduleNames.length` 的数组，该数组的每个元素都为以下之一：
- 具有非空的 `resolveFileName` 属性的 `ResolvedModule` 的实例 - 用来获取 `moduleNames` 数组所对应的名称
- `undefined` - 如果无法获取模块名

您可以通过调用 `resolveModuleName` 来介入标准模块解析过程。

> resolveModuleName(moduleName: string, containingFile: string, options: CompilerOptions, moduleResolutionHost: ModuleResolutionHost): ResolvedModuleNameWithFallbackLocations

此函数返回一个对象，该对象存储模块解析的结果（ `resolvedModule` 属性的值）以及在被视为候选文件的文件名列表。

```ts
import * as ts from "typescript";
import * as path from "path";

function createCompilerHost(options: ts.CompilerOptions, moduleSearchLocations: string[]): ts.CompilerHost {
  return {
    getSourceFile,
    getDefaultLibFileName: () => "lib.d.ts",
    writeFile: (fileName, content) => ts.sys.writeFile(fileName, content),
    getCurrentDirectory: () => ts.sys.getCurrentDirectory(),
    getDirectories: path => ts.sys.getDirectories(path),
    getCanonicalFileName: fileName =>
      ts.sys.useCaseSensitiveFileNames ? fileName : fileName.toLowerCase(),
    getNewLine: () => ts.sys.newLine,
    useCaseSensitiveFileNames: () => ts.sys.useCaseSensitiveFileNames,
    fileExists,
    readFile,
    resolveModuleNames
  };

  function fileExists(fileName: string): boolean {
    return ts.sys.fileExists(fileName);
  }

  function readFile(fileName: string): string | undefined {
    return ts.sys.readFile(fileName);
  }

  function getSourceFile(fileName: string, languageVersion: ts.ScriptTarget, onError?: (message: string) => void) {
    const sourceText = ts.sys.readFile(fileName);
    return sourceText !== undefined
      ? ts.createSourceFile(fileName, sourceText, languageVersion)
      : undefined;
  }

  function resolveModuleNames(
    moduleNames: string[],
    containingFile: string
  ): ts.ResolvedModule[] {
    const resolvedModules: ts.ResolvedModule[] = [];
    for (const moduleName of moduleNames) {
      // 尝试使用标准解析
      let result = ts.resolveModuleName(moduleName, containingFile, options, {
        fileExists,
        readFile
      });
      if (result.resolvedModule) {
        resolvedModules.push(result.resolvedModule);
      } else {
        // 检查回调位置，为了简单起见，我们认为模块是由 '.d.ts' 文件表示的
        for (const location of moduleSearchLocations) {
          const modulePath = path.join(location, moduleName + ".d.ts");
          if (fileExists(modulePath)) {
            resolvedModules.push({ resolvedFileName: modulePath });
          }
        }
      }
    }
    return resolvedModules;
  }
}

function compile(sourceFiles: string[], moduleSearchLocations: string[]): void {
  const options: ts.CompilerOptions = {
    module: ts.ModuleKind.AMD,
    target: ts.ScriptTarget.ES5
  };
  const host = createCompilerHost(options, moduleSearchLocations);
  const program = ts.createProgram(sourceFiles, options, host);

  /// do something with program...
}
```

## 创建和打印一个 TypeScript AST

TypeScript 具有工厂函数和可以结合使用的打印 API

- 该工厂允许您以 TypeScript 的 AST 格式生成新的树节点。
- 打印机可以采用一棵已有的树（由 `createSourceFile` 生成或由工厂函数生成的树），并生成输出字符串。

这是一个利用两者来产生阶乘函数的示例：

```ts
import * as ts from "typescript";

function makeFactorialFunction() {
  const functionName = ts.factory.createIdentifier("factorial");
  const paramName = ts.factory.createIdentifier("n");
  const parameter = ts.factory.createParameterDeclaration(
    /*decorators*/ undefined,
    /*modifiers*/ undefined,
    /*dotDotDotToken*/ undefined,
    paramName
  );

  const condition = ts.factory.createBinaryExpression(paramName, ts.SyntaxKind.LessThanEqualsToken, ts.factory.createNumericLiteral(1));
  const ifBody = ts.factory.createBlock([ts.factory.createReturnStatement(ts.factory.createNumericLiteral(1))], /*multiline*/ true);

  const decrementedArg = ts.factory.createBinaryExpression(paramName, ts.SyntaxKind.MinusToken, ts.factory.createNumericLiteral(1));
  const recurse = ts.factory.createBinaryExpression(paramName, ts.SyntaxKind.AsteriskToken, ts.factory.createCallExpression(functionName, /*typeArgs*/ undefined, [decrementedArg]));
  const statements = [ts.factory.createIfStatement(condition, ifBody), ts.factory.createReturnStatement(recurse)];

  return ts.factory.createFunctionDeclaration(
    /*decorators*/ undefined,
    /*modifiers*/ [ts.factory.createToken(ts.SyntaxKind.ExportKeyword)],
    /*asteriskToken*/ undefined,
    functionName,
    /*typeParameters*/ undefined,
    [parameter],
    /*returnType*/ ts.factory.createKeywordTypeNode(ts.SyntaxKind.NumberKeyword),
    ts.factory.createBlock(statements, /*multiline*/ true)
  );
}

const resultFile = ts.createSourceFile("someFileName.ts", "", ts.ScriptTarget.Latest, /*setParentNodes*/ false, ts.ScriptKind.TS);
const printer = ts.createPrinter({ newLine: ts.NewLineKind.LineFeed });

const result = printer.printNode(ts.EmitHint.Unspecified, makeFactorialFunction(), resultFile);
console.log(result);
```

## 使用类型检查器

在此示例中，我们将遍历 AST 并使用检查器去序列化类的信息。我们将使用类型检查器来获取符号和类型信息，同时获取已导出的类，其构造函数和各个构造函数参数的 JSDoc 注释。

```ts
import * as ts from "typescript";
import * as fs from "fs";

interface DocEntry {
  name?: string;
  fileName?: string;
  documentation?: string;
  type?: string;
  constructors?: DocEntry[];
  parameters?: DocEntry[];
  returnType?: string;
}

/** Generate documentation for all classes in a set of .ts files */
function generateDocumentation(
  fileNames: string[],
  options: ts.CompilerOptions
): void {
  // Build a program using the set of root file names in fileNames
  let program = ts.createProgram(fileNames, options);

  // Get the checker, we will use it to find more about classes
  let checker = program.getTypeChecker();
  let output: DocEntry[] = [];

  // Visit every sourceFile in the program
  for (const sourceFile of program.getSourceFiles()) {
    if (!sourceFile.isDeclarationFile) {
      // Walk the tree to search for classes
      ts.forEachChild(sourceFile, visit);
    }
  }

  // print out the doc
  fs.writeFileSync("classes.json", JSON.stringify(output, undefined, 4));

  return;

  /** visit nodes finding exported classes */
  function visit(node: ts.Node) {
    // Only consider exported nodes
    if (!isNodeExported(node)) {
      return;
    }

    if (ts.isClassDeclaration(node) && node.name) {
      // This is a top level class, get its symbol
      let symbol = checker.getSymbolAtLocation(node.name);
      if (symbol) {
        output.push(serializeClass(symbol));
      }
      // No need to walk any further, class expressions/inner declarations
      // cannot be exported
    } else if (ts.isModuleDeclaration(node)) {
      // This is a namespace, visit its children
      ts.forEachChild(node, visit);
    }
  }

  /** Serialize a symbol into a json object */
  function serializeSymbol(symbol: ts.Symbol): DocEntry {
    return {
      name: symbol.getName(),
      documentation: ts.displayPartsToString(symbol.getDocumentationComment(checker)),
      type: checker.typeToString(
        checker.getTypeOfSymbolAtLocation(symbol, symbol.valueDeclaration!)
      )
    };
  }

  /** Serialize a class symbol information */
  function serializeClass(symbol: ts.Symbol) {
    let details = serializeSymbol(symbol);

    // Get the construct signatures
    let constructorType = checker.getTypeOfSymbolAtLocation(
      symbol,
      symbol.valueDeclaration!
    );
    details.constructors = constructorType
      .getConstructSignatures()
      .map(serializeSignature);
    return details;
  }

  /** Serialize a signature (call or construct) */
  function serializeSignature(signature: ts.Signature) {
    return {
      parameters: signature.parameters.map(serializeSymbol),
      returnType: checker.typeToString(signature.getReturnType()),
      documentation: ts.displayPartsToString(signature.getDocumentationComment(checker))
    };
  }

  /** True if this is visible outside this file, false otherwise */
  function isNodeExported(node: ts.Node): boolean {
    return (
      (ts.getCombinedModifierFlags(node as ts.Declaration) & ts.ModifierFlags.Export) !== 0 ||
      (!!node.parent && node.parent.kind === ts.SyntaxKind.SourceFile)
    );
  }
}

generateDocumentation(process.argv.slice(2), {
  target: ts.ScriptTarget.ES5,
  module: ts.ModuleKind.CommonJS
});
```

这样子使用它:

```shell
tsc docGenerator.ts --m commonjs
node docGenerator.js test.ts
```

输入文件如下:
```ts
/**
 * Documentation for C
 */
class C {
    /**
     * constructor documentation
     * @param a my parameter documentation
     * @param b another parameter documentation
     */
    constructor(a: string, b: C) { }
}
```

那么我们就会能到输出值如下：
```json
[
    {
        "name": "C",
        "documentation": "Documentation for C ",
        "type": "typeof C",
        "constructors": [
            {
                "parameters": [
                    {
                        "name": "a",
                        "documentation": "my parameter documentation",
                        "type": "string"
                    },
                    {
                        "name": "b",
                        "documentation": "another parameter documentation",
                        "type": "C"
                    }
                ],
                "returnType": "C",
                "documentation": "constructor documentation"
            }
        ]
    }
]
```

> 原文链接：https://github.com/microsoft/TypeScript/wiki/Using-the-Compiler-API