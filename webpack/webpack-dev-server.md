#### webpack-dev-server

初始环境变量配置完毕后，会发现，单纯使用webpack以及它的命令行工具来进行开发调试的效率并不高。现在修改项目之后，多了一步打包操作，才能看到更新后的结果，刷新页面才有效，有没有更简便的方法呢？

webpack社区已经为我们提供了一个便捷的本地开发工具-webpack-dev-server：

```
  npm install webpack-dev-server --save-dev
  --save-dev参数将webpack-dev-server作为工程的devDependencies(开发环境依赖)记录到package.json中。这样做是因为webpack-dev-server仅仅在本地开发才会用到，在生产环境并不需要它。假如工程上线时需要进行依赖安装，就可以通过npm install --production 过滤掉devDependencies中的冗余模块，从而加快安装和发布的速度。
```
