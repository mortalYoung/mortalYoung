# DOM 和 BOM

DOM => Document Object Model 文档对象模型
BOM => Browser Object Model 浏览器对象模型

## Points

DOM ==(core)=> Document
BOM ==(core)=> Window

Window 具有 navigator, history 等对象 => Window 主要负责浏览器的交互行为
Document 具有 links, forms 等对象 => Document 主要负责页面节点的行为

Window 具有 document 属性 => Window 是 Document 的父级 => BOM 是 DOM 的父级

DOM 树是指 浏览器会将收到的 HTML 标签解析成树状结构, 每一个标签会被解析成一个对象, 标签的属性比如 href, id, class 都会被解析成对象的属性
