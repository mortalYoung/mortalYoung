# bundler 选择

由于 Molecule 2.0 的开发，需要选择一个打包器来支持，该打包器需满足如下需求：
1. 面向 npm package 的
2. bundless
3. 编译 sass 样式
4. alias
5. 针对其他文件，复制操作

# 对比

| 名称              | 描述                                                 | 备注                                   |
| ----------------- | ---------------------------------------------------- | -------------------------------------- |
| Vite              | 面向 Web Application 的 bundless 构建工具            | 不符合 lirary                          |
| father            | 面向 npm package 的 bundless 构建工具                | 无法自定义插件，无法解决样式文件的问题 |
| rspack（webpack） | 面向 Web Application 的 bundler                      | 不符合 bundless                        |
| esbuild           | 支持 bundless，支持自定义插件                        | 没有一整套的解决方案                   |
| swc               | Speedy Web Compiler，面向 Web Application 的构建工具 | 不符合 npm package 的要求              |
| parcel            | 面向 Web Application 的构建工具                      | 不符合 npm pacakge 的要求              |
| rollup            | 面向 npm package 的构建工具                          | 无法 bundless                          |
| tsc               | 支持 bundless                                        | 没有一整套的解决方案，且效率不如 esbuild                                       |
