# 链接方式

## 前言

- 在修复 `dumi` 问题的时候遇到了用 `symlink` 创建的文件但是 `existsSync` 是 `false` 的情况
- 在查询了相关的文档之后也没有遇到很清楚的解决方案
- 所以在此记录一下解题思路

## 问题复现

### 环境搭建

```bash
$ mkdir test & cd test
# 借用一下 `symlink-dir` 这个包来实现 `JUNCTION` 文件
$ yarn add symlink-dir --save-dev
```

然后在 `test` 目录下新建 `index.js`

相关 api 自行查阅

[symlink-dir](https://github.com/zkochan/symlink-dir)

[fs.symlink](http://nodejs.cn/api/fs.html#fs_fs_symlink_target_path_type_callback)

```js
// index.js
const symlinkDir = require("symlink-dir");
const path = require("path");
const fs = require("fs");

// 这里的意思是把 path.resolve(__dirname, "./symlink") 指向 path.resolve(__dirname, "index.js")
symlinkDir(
  path.resolve(__dirname, "index.js"),
  path.resolve(__dirname, "./symlink")
).then((res) => {
  console.log("res:", res);
});
```

执行这段文件可以看到在当前 `test` 目录下生成了 `symlink` 文件

![image](https://raw.githubusercontent.com/mortalYoung/mortalYoung/master/images/symlink-menu.png)

但是当你去点击该文件的时候会遇到 `Error` 信息如下

```
无法打开“symlink”: 无法读取文件'd:\projects\test\symlink' (EntryNotFound (FileSystemError): Error: ENOENT: no such file or directory, open 'd:\projects\test\symlink')。
```

或者你在文件资源管理器中打开该文件的时候同样会报出错误提示

```
位置不可用
× 无法访问 D:\projects\test\symlink
目录名称无效
```

到这里我以为是创建 `symlink` 失败了, 在查阅了大量资料后依旧一无所获

### `symlink` 文件查看

后来我了解到在 `Linux` 下用 `ll` 命令查看的时候会有类似于下面的信息显示

```bash
$ ll symlink
lrwxr-xr-x ... target -> path
```

于是我在 `windows` 下执行 `dir` 命令去查看 `symlink` 文件是否正常 我获得了如下信息

```base
D:\projects\test
λ dir
...
2020/08/22  21:42    <DIR>          .
2020/08/22  21:42    <DIR>          ..
2020/08/22  22:48               595 index.js
2020/08/22  21:37    <DIR>          node_modules
2020/08/22  21:37                61 package.json
2020/08/22  21:42    <JUNCTION>     symlink [D:\projects\test\index.js] 2020/08/
...
```

显然可以看到该文件是存在的，并且在我认为这是正常的

遇到我推翻了我前面认为的 `symlink` 生成符号链接失败的推论

### 围观源码

那么接下来我认为是 `existsSync` 的问题，于是我去查看了 👉 [`fs.existsSync`](https://github.com/nodejs/node/blob/master/lib/fs.js#L240) 的源码

摘抄出来如下

```js
function existsSync(path) {
  ...
  binding.access(nPath, F_OK, undefined, ctx);

  // In case of an invalid symlink, `binding.access()` on win32
  // will **not** return an error and is therefore not enough.
  // Double check with `binding.stat()`.
  if (isWindows && ctx.errno === undefined) {
    binding.stat(nPath, false, undefined, ctx);
  }

  return ctx.errno === undefined;
}
```

可以看到先用 `access` 判断了一次文件是否存在，然后如果再用 `stat` 来 `Double check`

先试试看 `access` 是否通过

```js
// index.js
...
fs.access(path.resolve(__dirname, "./symlink"), fs.constants.F_OK, (err) => {
  console.log(`${err ? "不存在" : "存在"}`);
});
...
```

得到的结果是 `存在`，即没有 `err` 信息

那么在上面的 `if` 条件判断中 `ctx.errno === undefined` 会是真，从而进行 `stat` 的判断

```js
// index.js
...
fs.stat(path.resolve(__dirname, "./symlink"), (err) => {
  console.log(err);
});
...
```

返回结果如下

```base
D:\projects\test
λ node index.js
[Error: ENOENT: no such file or directory, stat 'D:\projects\test\symlink'] {
  errno: -4058,
  code: 'ENOENT',
  syscall: 'stat',
  path: 'D:\\projects\\test\\symlink'
}
```

### 得出结论

到这里得出结论 `existsSync` 返回 `false` 的原因是因为在 `existsSync` 中用 `stat` 进行 `Double check` 的时候返回了错误信息

从注释中可以看出 `symlink` 生成的文件是 `an invalid symlink` 这是这一段 `if` 特地去规避的

**但是但是**
在围观了 `exists` 并且实测以后，发现同步的 `exists` 是可以 `return true` 的

**不过，`exists` 是要被废弃的方法**

**然而，`exists` 其实就是封装了的 `access`**

## 尾声

所以暂时明白了如果需要的话 在 `windows` 下用 `fs.access` 来判断 `symlink` 是否生成

不过按照官方注释来看，生成的是一个 `invalid symlink`

但是我不认为这是一个 `invalid symlink` 

因为我用 `mklink /J symlint index.js` 命令创建的 `symlint` 文件和用 `fs.symlint` 命令创建的文件所表现出来的形式是一样的

所以我不知道 `node` 源码里提到的 `invalid symlink` 到底是不是指的这种

如果你凑巧看到这边文档，并且你也知道在 `windows` 下生成的 `valid symlink` 是怎么样的

请告诉你是怎么实现的

以上。
