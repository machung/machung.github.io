# Build Vue 2.x app with webpack from scratch

1. 建立 app 目錄: `mkdir vueapp`. 以下簡稱根目錄.
1. 進入目錄, 執行 `npm init` 建立 __package.json__ 檔案.
1. 安裝 vue, webpack. 執行 `npm i -D vue vue-loader vue-template-compiler css-loader webpack webpack-cli`.
1. 建立 __src__ 目錄存放原始碼.
1. 進入 __src__ 目錄, 建立 __app.vue__ 檔案.
1. 編輯 __app.vue__ 內容如下:

    ```html
    <template>
        <div id="test">{{text}}</div>
    </template>
    <script>
        export default {
            data () {
                text: 'Hello World!'
            }
        }
    </script>
    <style>
    #test{
        font-size:12px;
        color:green;
    }
    </style>
    ```

1. 進入根目錄, 建立 __webpack.config.js__ 檔案.
1. 編輯 __webpack.config.js__ 如下:

    ```js
    const path = require('path')

    module.exports = {
        entry:  path.join(__dirname, 'src/index.js'), // webpack4 預設路徑就是 ./src/index.js
        output: {
            filename: 'bundle.js',
            path: path.join(__dirname, 'dist') // webpack4 預設路徑就是 ./dist
        }
    }
    ```

    > `__dirname` 指的就是這個檔案所在的目錄路徑.

1. 進入 __src__ 目錄, 建立 __index.js__ 檔案並編輯:

    ```js
    import Vue from 'vue'
    import App from './app.vue'

    const root = document.createElement('div')
    document.body.appendChild(root)

    new Vue({
        render: (h) => h(App)
    }).$mount(root)
    ```

1. 進入根目錄, 編輯 __package.json__, 加入一段 scripts 設定:

    ```json
    "scripts": {
        "build": "webpack --config webpack.config.js --mode production"
    }
    ```

1. 執行 `npm run build`. 如果 __vue-loader__ 的版本是15以上, 則會出現下列訊息:

    > vue-loader was used without the corresponding plugin. Make sure to include VueLoaderPlugin in your webpack config.

    vue-loader 從15版之後, 會需要在 [webpack.config.js 中設定 plugins 才行](https://vue-loader.vuejs.org/migrating.html#a-plugin-is-now-required). 因此需要修改 webpack.config.js:

    ```js
    const VueLoaderPlugin = require('vue-loader/lib/plugin')

    module.exports = (env, argv) => {
        const devMode = argv['mode'] !== 'production' // 取得 webpack --mode 的值來判斷是development還是production mode

        return {
            mode: devMode ? 'development' : 'production',
            // ...
            plugins: [
                new VueLoaderPlugin()
            ]
        }
    }
    ```

1. 安裝 __webpack-dev-server__: `npm i -D webpack-dev-server`
1. 修改 __package.json__, 加入 script:

    ```json
    "scripts": {
        // ...
        "start": "webpack-dev-server --mode development --open --hot"
    }
    ```

1. 設定 webpack modules
    1. .vue

        ```js
        module: {
            rules: [
                {
                    test: /.vue$/,
                    loader: 'vue-loader'
                },
            ]
        ```

    2. .js

        ```js
        {
            test: /\.js$/,
            loader: 'babel-loader',
            exclude: file => (
                /node_modules/.test(file) &&
                !/\.vue\.js/.test(file)
            )
        }
        ```

        使用 __babel-loader__ 需要先安裝套件: `npm i -D @babel/core babel-loader @babel/preset-env`

        在根目錄中建立 __.babelrc__ 檔案:

        ```json
        {
            "presets": [
                "@babel/preset-env"
            ]
        }
        ```

    3. .sass, .css, scss

        ```js
        {
            test: /\.(sa|sc|c)ss$/,
            use: [
                devMode ? 'style-loader' : MiniCssExtractPlugin.loader,
                'css-loader',
                'postcss-loader',
                'sass-loader'
            ]
        }
        ```

        使用這些 css-like loaders 需要先安裝套件: `npm i -D mini-css-extract-plugin css-loader style-loader postcss-loader postcss-import sass-loader node-sass`

        修改 __webpack.config.js__:

        ```js
        // ...
        const MiniCssExtractPlugin = require("mini-css-extract-plugin")

        module.exports = (env, argv) => {
            // ...
            plugins: [
                // ...
                new MiniCssExtractPlugin({
                    filename: devMode ? '[name].css' : '[name].[hash].css',
                    chunkFilename: devMode ? '[id].css' : '[id].[hash].css'
                })
            ]
        }
        ```

        在根目錄建立 __postcss.config.js__ 檔案:

        ```js
        module.exports = ({ file, options, env }) => ({
            parser: file.extname === '.sss' ? 'sugarss' : false,
            plugins: {
                'postcss-import': { root: file.dirname },
                'postcss-preset-env': options['postcss-preset-env'] ? options['postcss-preset-env'] : false,
                'cssnano': env === 'production' ? options.cssnano : false
            }
        })
        ```

1. webpack執行build時, 我們會需要其協助我們產生一個網頁檔並自動引用產生出來的js與css檔案. `html-webpack-plugin`套件可以協助我們達成.
    1. 先安裝套件: `npm i -D html-webpack-plugin`.
    2. 加入plugins:

    ```js
    // ...
    const HtmlWebPackPlugin = require("html-webpack-plugin")

    module.exports = (env, argv) => {
        // ...
        plugins: [
            // ...
            new HtmlWebPackPlugin()
        ]
    }
    ```

1. 當每次執行 __build__ 時, 預設output目錄為 __dist__, 而此目錄中的檔案會有殘留的問題. 因此我們需要設定讓webpack每次build時都將output目錄清除以確保不會有多餘的檔案殘留. 要做到此功能, 需要安裝 `clean-webpack-plugin` 這個套件來協助達成此目的.
    1. 執行 `npm i -D clean-webpack-plugin` 安裝套件
    2. 修改 __webpack.config.js__:

        ```js
        const CleanWebpackPlugin = require('clean-webpack-plugin')
        // ...
        plugins: [
            new CleanWebpackPlugin(['dist']),
            // ...
        ]
        ```

1. 在 __index.js__ 中加入 router
    1. 執行 `npm i -S vue-router` 安裝 __vue-router__ 套件.
    1. 編輯 __index.js__

        ```js
        // ...

        // Todo, Settings, About 分別為三個不同的 vue 頁面
        import Todo from './components/TodoList.vue'
        import Settings from './components/Settings.vue'
        import About from './components/About.vue'

        import VueRouter from 'vue-router'

        Vue.use(VueRouter)

        const routes = [
            { path: '/todo', component: Todo},
            { path: '/settings', component: Settings},
            { path: '/about', component: About},
        ]

        const router = new VueRouter(
            {
                routes // 屬性名稱與變數名稱相同的簡寫, 同等於 routes: routes
            }
        )

        // ...

        new Vue({
            render: h => h(App),
            router
        }).$mount(root)
        ```
