---

layout: post
title: vue-cli + webpack + bootstrap 模板项目生成步骤 
date: 2018-03-13 16:06:24.000000000 +09:00
tags: 前端

---

## 创建 vue 项目

首先根据 `vue` 官方教程，快速创建配置好的模板项目。

```shell
# 全局安装 vue-cli
$ npm install --global vue-cli
# 创建一个基于 webpack 模板的新项目，my-project 名称可自行修改，不能有中文
$ vue init webpack my-project
```

安装时，会提示要不要安装其他一些插件。为简单起见，暂时先不安装这些额外的插件。

```Shell
? Project name vue-js (回车)
? Project description A Vue.js project (回车)
? Author ... (回车)
? Vue build standalone (回车)
? Install vue-router? No (输入n再回车)
? Use ESLint to lint your code? No (输入n再回车)
? Set up unit tests No (输入n再回车)
? Setup e2e tests with Nightwatch? No (输入n再回车)
? Should we run `npm install` for you after the project has been created? (recom
mended) npm (回车)
```

下载创建完成后，运行以下命令。

```shell
# 安装依赖
$ cd my-project
$ npm install
# 启动
$ npm run dev
```

在浏览器地址栏输入 http://localhost:8080 然后回车，看到以下页面代表 `vue` 模板项目安装成功。

![](/assets/images/2018/vue-webpack-bootstrap-project-create-step-1.png)

## 安装和配置 jquery

因为 `bootstrap` 是依赖 `jquery` 的，所以必须先安装 `jquery`。

```Shell
npm install jquery --save-dev
```

用某个 IDE 打开项目，打开 build/webpack.base.conf.js 文件，在文件最上面，`'use strict'` 后面添加以下代码。

```javascript
const webpack = require('webpack')
```

然后在 `module.exports` 配置对象的末尾添加 `plugins` 配置，记得在前面加`逗号`！！！

```javascript
  plugins: [
    new webpack.ProvidePlugin({
      $: "jquery",
      jQuery: "jquery",
      "windows.jQuery": "jquery"
    })
  ]
```

## 安装 bootstrap

运行命令安装。

```shell
npm install bootstrap@3 --save-dev 
```

安装完成后，打开项目入口 main.js，引用 `bootstrap` 相关文件。

```javascript
import 'bootstrap/dist/css/bootstrap.min.css'
import 'bootstrap/dist/js/bootstrap.min.js'
```

引用了之后，在 HelloWorld.vue 添加以下代码，检验是否 `bootstrap` 安装成功。

```html
<table class="table table-striped">
  <thead>
    <tr>
      <th>#</th>
      <th>Header</th>
      <th>Header</th>
      <th>Header</th>
      <th>Header</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>1,001</td>
      <td>Lorem</td>
      <td>ipsum</td>
      <td>dolor</td>
      <td>sit</td>
    </tr>
    <tr>
      <td>1,002</td>
      <td>amet</td>
      <td>consectetur</td>
      <td>adipiscing</td>
      <td>elit</td>
    </tr>
    <tr>
      <td>1,003</td>
      <td>Integer</td>
      <td>nec</td>
      <td>odio</td>
      <td>Praesent</td>
    </tr>
    <tr>
      <td>1,003</td>
      <td>libero</td>
      <td>Sed</td>
      <td>cursus</td>
      <td>ante</td>
    </tr>
  </tbody>
</table>
```

能看到下面的效果，并且没有报错就代表安装成功！！

![](/assets/images/2018/vue-webpack-bootstrap-project-create-step-2.png)

到此，一个 `vue + bootstrap` 项目就初始化完成了。