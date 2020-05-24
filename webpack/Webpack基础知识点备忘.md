### webpack配置文件拆分

1. 通用配置文件
2. dev.config.js
3. prod.config.js

他们之间桥梁是 `webpack-merge`

### 定义环境变量

webpack.definePlugin

### js打包

Babel-loader Babel.rc @babel/preset-env

### css打包

loader的执行顺序 **从右到左**

### img打包

- dev 模式 使用 file-loader
- Prod 模式 使用 url-loader ，可以添加更多的配置
  - limit
  - outputPath
  - publicPath

### 打包前清空打包目录

cleanWebpackPlugin 默认清空 output 的 path 指定的目录

### 多页面打包

1. Entry 对应多入口
2. output打包出的文件名是动态 [name] 参数，与entry进行对应
3. HtmlWebpackPlugin 多入口匹配多html插件处理
   1. 指定要加载的页面入口 js文件
   2. 通过 chunks: ['index'] 配置

### prod环境css文件抽离

MiniCssExtractPlugin

### Optimization webpack单独配置项

- minimizer: [new TerserJsPligin(), new OptimizeCssAssetsPlugin()]

### 抽离公共代码和第三方的代码

```javascript
optimization: {
 	splitChunks: {
    chunks: 'all', // initial 对于异步导入的文件不处理，async 只对异步导入的文件进行处理，all 都处理
    cacheGroups: { // 缓存分组
      // 第三方模块
      vendor: {
        name: 'vendeor', // 抽离的chunk名称
        priority: 1, // 命中多个缓存组配置，则比较抽离的优先级
        test: /node_modules/,
        minSize: 0, // 小于 minSize 则不再进行抽离
        minChunks: 1 // 最少复用几次
      },
      // 公共模块
      common: {
        name: 'common', // 抽离的chunk名称
        priority: 0, // 抽离的优先级
        minSize: 0, // 公共模块的打下限制
        minChunks: 2 // 公共模块最少复用几次
      }
    }
  } 
}
```

其中多入口 HtmlWebpackPlugin 的 chunks 引用需要注意

### 资源懒加载

```javascript
import(/* webpackChunkName: "data" */, './data.js').then(() => {});
```

### Module、chunk、bundle 的区别

1. module 各个源码文件，webpack中一切皆模块
2. chunk 多模块打包合成的一个集合
3. bundle 最终的输出文件，对chunk的集成

### webpack性能优化

1. 优化打包构建速度

   1. 优化babel-loader

      1. 开启缓存 cacheDirectory
      2. 明确打包范围 include

   2. IgnorePlugin

      1. 忽略一个包中某些文件的打包 new webpack.IgnorePlugin(/^\.\/locale$/, /moment$/)

   3. 多进程打包 happyPack

      1. Happypack/loader?id=babel

   4. ParallelUglifyPlugin 多进程压缩js

   5. 自动刷新（热更新）

      1. HotModuleReplacementPlugin

   6. DllPlugin 随着webpack的升级，效果一般

      体积大、不常升级 react、react-dom，一次打包多次使用

      1. 生成dll库文件
      2. 生成dll的索引文件
      3. 通过reference引用到dll文件
      4. 只用于开发环境

2. 优化产出代码

   1. 小图片base64编码
   2. Bundle 加 hash 
   3. 懒加载
   4. 提取公共代码
   5. IgnorePlugin
   6. Scope Hosting
      1. 代码体积更小
      2. 创建函数作用域更是
   7. Tree-Shaking
      1. mode=production 自动开启
      2. 必须使用 ES6 module 才能生效，commonjs无效，原因在于 ES6 import 为静态引入，可以进行代码静态分析





