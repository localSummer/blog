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
   6. production 模式
   7. Scope Hosting
      1. 代码体积更小
      2. 创建函数作用域更是
   8. Tree-Shaking
      1. mode=production 自动开启
      2. 必须使用 ES6 module 才能生效，commonjs无效，原因在于 ES6 import 为静态引入，可以进行代码静态分析

### babel-polyfill

1. corejs、regenerator 的集合

2. babel 7.4后弃用 babel-polyfill，推荐直接使用 corejs、regenerator

3. babel 只解析语法，对于新的 API 并不处理、同事也不处理模块化（模块化使用webpack进行处理）
4. 垫片的按需加载

```json
{
  "presets": [
    [
      "@babel/preset-env",
      {
        "useBuiltIns": 'usage',
        "corejs": 3
      }
    ]
  ],
  "plugins": [
    [
      // 不会污染全局变量
      // 多次使用只会打包一次
      // 依赖统一按需引入,无重复引入,无多余引入
      "@babel/plugin-transform-runtime", // 默认配置即可
      {
        "absoluteRuntime": false,
        "corejs": 3,
        "helpers": true,
        "regenerator": true, // 避免污染全局域
        "useEsModules": false,
        "moduleName": "babel-runtime" // 默认值
      }
    ]
  ]
}
```

5. 造成全局环境的污染，@babel/runtime 对 API 名字进行转换，配合 `@babel/plugin-transform-runtime` 使用

### 为何进行打包和构建

1. 体积更小，加载更快
2. 编译高级语言和语法（TS、ES6+）
3. 兼容性和错误检查（polyfill、eslint）
4. 统一高效的研发流程
5. 统一的构建流程和产出标准
6. 集成公司构建规范

### loader和plugin的区别

loader模块转换器、plugin扩展插件

### 为何 Proxy 不能被polyfill

class 可以用 function 模拟、Promise 可以被 callback 模拟

Proxy的功能用 Object.defineProperty 无法模拟，对象的属性新增和删除无法被拦截到，数组也无法拦截















