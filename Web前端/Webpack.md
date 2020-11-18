## 概述
https://www.webpackjs.com/

Webpack为JavaScript以及相关的其他web前端资源提供`模块化`和`打包`的功能：
* 模块化：支持JavaScript的CommonJS、ES6等，css/sass/less文件中的 @import 语句，及其其他
* 打包：将各种资源打包，打包过程还可以执行例如JS合并为一个文件、ES6转换、sass转换等

Webpack依赖于Node

## 基本使用
1. 不使用配置文件的情况下：
```
webpack <entry> [<entry>] -o <output>
```
例如项目结构如下：
```
.
├── dist
├── index.html
└── src
    ├── index.js
    ├── index2.js
    └── others.js
```
```
webpack src/index.js -o dist/bundle.js
```
这个-o似乎可以省略

2. 使用配置文件
可以在当前目录中写一个配置文件webpack.config.js，这样就可以直接执行webpack命令，具体的行为在配置文件指定，比如入口文件和输出文件。
```
webpack [--config webpack.config.js]
```
配置文件默认名称为webpack.config.js，如果使用其他名称，那么需要明确指定

----------

实际项目中，一般会在一个npm管理的项目中使用webpack
```
npm init -y
npm install webpack webpack-cli --save-dev
```
然后结合npm的package.json文件和webpack一起使用

## Loader
webpack默认只支持处理JS文件，如果在JS文件中使用require引入css文件
```
//js

import './style.css';
```
js文件中依赖了css文件，打包过程是无法完成的，如果要处理CSS，那么需要使用CSS对应的css-loader

不过，css-loader只负责加载css文件，如果要使css样式生效，还要使用style-loader

## Plugin
Loader用于某些类型文件的转换，而Plugin是对webpack本身的扩展，比如给每个JS文件添加License、JS代码丑化