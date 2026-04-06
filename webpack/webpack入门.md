# webpack 入门

## 为什么需要 webpack

在早期前端开发中：

问题：

- 浏览器只支持 script 引入
- 没有模块化
- 依赖关系难管理
- HTTP 请求过多

例如：

```javascript
<script src="jquery.js"></script>
<script src="util.js"></script>
<script src="app.js"></script>
```

## webpack 是什么？

Webpack 是一个 模块打包器（module bundler），用于将项目中的各种资源（JS、CSS、图片、字体等）当作 模块 进行处理，并最终打包成浏览器可运行的静态资源。

核心目标：

- 解决浏览器不支持模块的问题
- 构建依赖关系图
- 优化资源加载性能
- 统一前端工程构建流程

所以 webpack 有一句核心思想：

> Everything is Module





