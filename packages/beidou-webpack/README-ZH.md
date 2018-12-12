# beidou-webpack

北斗 webpack 开发插件

## 特性

- 编译 jsx/js
- 编译 less/scss/css
- 支持热替换(hmr)
- 快速重启
- 支持自定义配置

## Breaking Changes

- 配置项变更，

  自`v1.0.0`起，beidou-webpack 插件底层依赖从 `webpackDevMiddleware` 迁移到了 `WebpackDevServer`，配置方式更趋近于普通非同构前端项目。具体变更参看 [issue#21](https://github.com/alibaba/beidou/issues/21)

## Configuration

只在非生产环境下使用本插件，所以在你的项目上线之前必须先编译好，并把编译结果打包到线上环境。

- config/plugin.js:

```js
exports.webpack = {
  enable: true,
  package: 'beidou-webpack',
  env: ['local', 'unittest'],
};
```

- config/config.local.js **默认配置如下：**

```js
exports.webpack = {
  custom: {
    // configPath: 'path/to/webpack/config/file',
  },
  output: {
    path: './build',
    filename: '[name].js?[hash]',
    chunkFilename: '[name].js',
    publicPath: './build',
  },

  resolve: {
    extensions: ['.json', '.js', '.jsx'],
  },

  devServer: {
    contentBase: false,
    port: 6002,
    noInfo: true,
    quiet: false,
    clientLogLevel: 'warning',
    lazy: false,
    watchOptions: {
      aggregateTimeout: 300,
    },
    headers: { 'X-Custom-Header': 'yes' },
    stats: {
      colors: true,
      chunks: false,
    },
    publicPath: '/build',
    hot: true,
  },
};
```

配置项 `output`、`resolve`、`devServer` 和 [webpack](https://webpack.js.org) 中定义的配置项一致。注意：任何有效的 webpack 配置都可以而不仅仅是这三个配置项。对这些配置的修改会对插件默认生成的 webpack 配置生效。插件自身使用的配置在 `custom` 里面，比如，自定义 webpack 配置需要在 `'webpack.custom.configPath'` 里设置文件路径。

- **devServer.port**： 定义 webpack dev server 的监听端口。访问 webpack 资源时，仍然可以通过 Node 应用直接访问，插件内置代理中间件自动转发请求到 webpack dev server ，所以  通常情况下不需要关心这个端口号。如果有特殊需要，可以直接访问 webpack dev server (上述配置下的地址 http://localhost:6002/webpack-dev-server)。

- **devServer.contentBase**：必须设置为 `false`。任何非`false`的值都会  启用 webpack dev server 中的静态资源服务。这将导致所有发送到 Node 服务的请求都提前被 webpack 响应。

## Custom webpack configuration

**custom.configPath**: 先定义 webpack 配置文件路径

```js
// webpack.config.js
// 配置方案1:
module.exports = (app, defaultConfig, dev, target) => {
  return {
    ...defaultConfig,
    entry: {
      // your entry
    },
    module: {
      // your module
    },
    plugins: [
      // your plugins
    ],
    //something else to override
  };
};

// 配置方案2:
// 此方案将直接覆盖原有配置
module.exports = {
  output: {
    path: './build',
    filename: '[name].js?[hash]',
    chunkFilename: '[name].js',
    publicPath: './build',
  },

  resolve: {
    extensions: ['.json', '.js', '.jsx'],
  },

  devServer: {
    contentBase: false,
    port: 6002,
    noInfo: true,
    quiet: false,
    clientLogLevel: 'warning',
    lazy: false,
    watchOptions: {
      aggregateTimeout: 300,
    },
    headers: { 'X-Custom-Header': 'yes' },
    stats: {
      colors: true,
      chunks: false,
    },
    publicPath: '/build',
    hot: true,
  },
};

// 配置方案3:
// 更多配置方法，可参看单测用例
module.exports = (app, defaultConfig, dev, target) => {
  const factory = app.webpackFactory;
  factory.set('output',{
    {
      path: outputPath,
      filename: '[name].js?[hash]',
      chunkFilename: '[name].modify.js',
      publicPath: '/build/',
    }
  })
  // 修改默认插件配置
  factory.modifyPlugin('CommonsChunkPlugin',
    factory.getPlugin('CommonsChunkPlugin'), // 如不修改，可传null
    {
      name: 'vendor',
      filename: 'manifest.js',
    }
  );

  factory.addPlugin(
    webpack.optimize.UglifyJsPlugin,
    {
      compress: {
        warnings: false,
      }
    },
    'UglifyJsPlugin' , // 可空
  )

  factory.addRule({
    test: /\.jsx?$/,
    exclude: /node_modules/,
    use: {
      loader: 'babel-loader',
      options: {
        babelrc: false,
        presets: ['beidou-client'],
      },
    },
  })

  const factoryInProd = factory.env('prod');
  factoryInProd.addPlugin(new webpack.optimize.UglifyJsPlugin({
    compress: {
      warnings: false,
    },
  }))

  return factory.getConfig(); // 返回配置

};

```

- **app**: `BeidouApplication` 示例, 通常用来读取应用的全局配置项。

- **defaultConfig**: 插件生成的默认 webpack 配置，可用于覆盖使用。

- **dev**: 本地开发环境下为`true`，生产环境下为`false` .

- **target**: 默认为 `browser`，在运行 `beidou build` 时, 通过 `--target` 参数指定, 目前可用值包含 `browser` 和 `node`

## Entry

默认 webpack 配置中的 entry 是通过扫描 client 目录获得。

扫描规则受 [beidou-router](../beidou-router/) 中
`router.root` 和 `router.entry` 两个配置项影响。

- **router.root**: 进行扫描的根路径
- **router.entry**: entry 文件名称，文件名（不含后缀）匹配的文件才被视为可用的 entry

> **Notice**:  扫描的文件深度为 1 (  仅匹配 ${router.root}/${router.entry}.jsx 和 ${router.root}/${dir}/\${router.entry}.jsx )

## Building

**用法**: beidou-build [--target=browser][--dev]

## License

[MIT](LICENSE)
