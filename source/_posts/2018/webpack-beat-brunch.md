---
title: 使用webpack取代brunch
date: 2018-01-13 23:04:30
tags:
  - elixir
  - erlang
  - phoenix
  - webpack
categories:
  - 折腾
---

[phoenix][phx]选择了[brunch][brunch]作为静态资源的管理工具，一番折腾之后，发现这个东西确实独特，配置少，上手简单，编译快。但是随着使用的深入，它的不足也越来越明显，老旧，更新缓慢，各种资源都年久失修。为了让`brunch`配合`vue`使用，我甚至还花了几天时间去修复插件`vue-brunch`的bug，于是，就有了本文所进行的探索。

不过选择[webpack][webpack]并不是因为它多么的好，恰恰相反，我反而觉得它太过复杂，概念繁多，文档看似丰富其实犹抱琵琶。对于新手实在太不友好。不过鉴于功能和流行程度，目前处理复杂的前端资源，`webpack`还是不二的选择。

### 目标
> 实现一个最小化的webpack配置，功能上要能完全取代默认的brunch的配置。

webpack复杂配置的可怕大家都是有目共睹的，所以我希望这个配置能足够精简，仅仅完成phoenix创建项目后默认提供的那些功能就好，因为对于webpack，想扩展功能太容易了。

### 第一步
首先从`assets/package.json`开始，需要先卸载`devDependencies`下所有的包，全都换成`webpack`系列，命令就不列了，毕竟复制粘贴太烦人，只截取展示一下`package.json`吧：
```json
  "devDependencies": {
    "@babel/core": "^7.0.0-beta.37",
    "@babel/preset-es2015": "^7.0.0-beta.37",
    "babel-loader": "^8.0.0-beta.0",
    "copy-webpack-plugin": "^4.3.1",
    "css-loader": "^0.28.8",
    "extract-text-webpack-plugin": "^3.0.2",
    "file-loader": "^1.1.6",
    "style-loader": "^0.19.1",
    "uglifyjs-webpack-plugin": "^1.1.6",
    "webpack": "^3.10.0"
  }
```

> 这里我激进的采用了*babel 7*，相应的，*babel-loader*的版本需要是*8*


### 主要部分
然后，就是最主要的`webpack.config.js`了，文件内容如下：
```javascript
const path = require('path');
const webpack = require('webpack');
const ExtractTextPlugin = require('extract-text-webpack-plugin');
const CopyWebpackPlugin = require('copy-webpack-plugin');
const UglifyJSPlugin = require('uglifyjs-webpack-plugin');

config = {
  devtool: 'eval-cheap-module-source-map',
  entry: {
    app: ['./css/app.css', './js/app.js']
  },
  output: {
    path: path.resolve(__dirname, '../priv/static'),
    filename: 'js/[name].js',
  },
  module: {
    rules: [{
        test: /\.css$/,
        use: ExtractTextPlugin.extract({
          fallback: 'style-loader',
          use: {
            loader: 'css-loader',
            options: { url: false }
          }
        })
      }, {
        test: /\.js$/,
        exclude: /node_modules/,
        loader: 'babel-loader',
        options: {
          presets: ['@babel/preset-es2015']
        }
      }, {
        test: /\.(png|jpg|gif)$/,
        loader: 'file-loader',
        options: {
          name: '[path][name].[ext]'
        }
      }
    ]
  },
  plugins: [
    new ExtractTextPlugin('css/[name].css'),
    new CopyWebpackPlugin([{
      from: './static'
    }])
  ]
}

if (process.env.MIX_ENV === 'prod' || process.env.NODE_ENV === 'production') {
  config.devtool = 'source-map'
  config.plugins.concat([
    new UglifyJSPlugin({
      parallel: true,
      sourceMap: true
    }),
    new webpack.optimize.ModuleConcatenationPlugin()
  ])
}

module.exports = config
```

同时，还需要在*./css/app.css*中加上：

```css
@import "./phoenix.css"
```

> 可以看到这里使用了4个*loader*，分别处理css，js和图片三种静态资源，
> 开发环境中，使用了用于提取css的*ExtractTextPlugin*和用来拷贝静态资源的*CopyWebpackPlugin*，
> 针对生产环境，增加了生成*source map*的选项并使用*uglifyjs*对代码进行压缩

需要注意的是，这里使用了*./css/app.css*作为入口之一，也就意味着不必在js文件中加入`import`来引入css文件，不过这样一来，就需要在app.css文件中加入上面那一行代码来解决引入css的问题。还有一种无需修改代码的方法，即把`./css/phoenix.css`也加入`entry`中，不过我认为这样不够直观，并不是一个好的选择。

依然是引入css的问题，`phoenix.css`中会导入一些字体文件，这些文件并不存在而且也无引入的必要，所以配置`css-loader`，将选项`url`置为`false`，这样webpack在编译时就不会试图检查这些字体了。

### 最后
为了让静态资源在发生变化时phoenix能够及时响应，必须修改`config/dev.exs`，变动如下：
```diff
-  watchers: [node: ["node_modules/brunch/bin/brunch", "watch", "--stdin",
+  watchers: [node: ["node_modules/webpack/bin/webpack.js", "--stdin", "--color",
```

`phoenix`默认配置了一些`npm scripts`，为了命令行保持统一，`assets/package.json`相应做如下修改：
```diff
-    "deploy": "brunch build --production",
-    "watch": "brunch watch --stdin"
+    "deploy": "MIX_ENV=prod webpack -p --progress",
+    "build": "webpack --progress",
+    "watch": "webpack --stdin --progress"
```
> 对于为何不用`npm run`命令来运行watcher，官方在[这里](https://github.com/phoenixframework/phoenix/issues/1540)做出了解释

最后的最后，不要忘记删除`brunch-config.js`，至此，就大功告成了。

附上目录下文件的变动情况：
```bash
    deleted:    assets/brunch-config.js
    modified:   assets/css/app.css
    modified:   assets/package-lock.json
    modified:   assets/package.json
    new file:   assets/webpack.config.js
    modified:   config/dev.exs
```


[phx]: http://phoenixframework.org
[brunch]: http://brunch.io
[webpack]: https://webpack.js.org