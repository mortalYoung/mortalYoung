# Boot up VSCode in China

## Electron mirror

add electron_mirror to `.npmrc`
```diff
+ electron_mirror="https://npmmirror.com/mirrors/electron/"
```

## Synchronizing built-in extensions

Make an exmaple with `ms-vscode.js-debug`.

- First, find the repo in Github
- Then, download the SourceCode tar/zip
- unzip it and move it to `vscode/.build/builtInExtensions`
- rename it to ms-vscode.js-debug
