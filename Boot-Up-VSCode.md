# Boot up VSCode in China

## node-pty runtime error

Check the supported version of Python/clang.

Upgrade XCode or run `xcode-select --install`. Otherwise, install llvm and use alias to install

```shell
# -------------------------------- #
# For clang++
# -------------------------------- #
export CC=/opt/homebrew/opt/llvm/bin/clang
export CXX=/opt/homebrew/opt/llvm/bin/clang++
```

## vscode/ripgrep 403

switch to global mode in clash. Or proxy http.

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
