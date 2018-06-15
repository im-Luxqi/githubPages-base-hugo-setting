---
title: Angular4公共项目模块管理 
date: 2018-06-14
tags: ["基础杂记"]
---

从npm 本地仓库的创建====》angular4项目的打包====》其他angular4项目的下载调用

<!--more-->
## 一、npm本地化仓库Sinopia
#### 1.解决问题
&emsp;&emsp;有时不同项目之间存在依赖，如果不想把项目发布到npm社区的仓库，则需要有自己本地的仓库。本地仓库候选方案有：sinopia，cnpm和kappa。
#### 2.sinopia特点
①零配置安装。  
②使用文件系统作为存储，仅保存用户需要的包，如果本地仓库没有对应的包，则从指定的registry下载，默认为npmjs.org。  
#### 3.安装
Sinopia的安装比较简单，只需使用npm一条安装命令即可
```javascript
npm install -g sinopia
```
***在windows下直接执行这个命令会遇到一些问题：***  
&emsp;&emsp;①python没有安装或版本不对,MSBuild版本不对   
`gyp ERR! stack Error: Can't find Python executable "python", you can set the PYTHON env variable.`  
`MSBUILD : error MSB4132: The tools version "2.0" is unrecognized. Available too ls versions are "4.0"`  
```javascript
//该类错误发生在node-gyp在构建时未能找到所需版本的构建工具
//解决方法如下：
npm install --global --production windows-build-tools
npm config set msvs_version 2015 --global
npm config set python python2.7 --global
```
***致命错误；***  
&emsp;&emsp;在后期的操作中，发现sinopia有致命缺陷，下载某些字符时404，查阅后得知sinopia源码存在符号转换的bug。  
&emsp;&emsp;并且sinopia作者已经在三年前停止对sinopia进行维护和升级,所以出来了一个sinopia的fork，名字叫做Verdaccio，
然后由Verdaccio继续对sinopia进行更新和维护。
## 一、坑啊，使用Verdaccio来构建私有npm服务器
[Verdaccio的github介绍](https://github.com/verdaccio/verdaccio)
#### 1.安装
```javascript
npm install -g verdaccio
```
#### 2.启动
```javascript
C:\Users\Rextec>verdaccio
 warn --- config file  - C:\Users\Rextec\AppData\Roaming\verdaccio\config.yaml
 warn --- Plugin successfully loaded: htpasswd
 warn --- Plugin successfully loaded: audit
 warn --- http address - http://localhost:4873/ - verdaccio/3.1.1
```
***进入config配置一下***
```javascript	
# 国内url 设置成淘宝镜像(优先在本地仓库下载，没有自动转到淘宝下载)
# a list of other known repositories we can talk to
uplinks:
  npmjs:
    url: https://registry.npm.taobao.org/
```
#### 3.使用准备
***添加用户***  
```javascript
$npm adduser --registry http://localhost:4873/
Username: wxq
Password: wxq
Email: (this IS public) im_luxqi@163.com
```
***使用用户登录***  
```javascript
$npm login
Username: wxq
Password: wxq
Email: (this IS public) im_luxqi@163.com
Logged in as wxq on http://localhost:4873/.
```
## 二、npm发布项目
&emsp;&emsp;通过`npm install`命令可以从npm仓库下载依赖包,极大方面对依赖包管理,功能类似`maven`。同样,我们也可以
通过`npm publish`命令将自己编写的代码提交到npm仓库供其他模块引用。
#### 1.进入需要打包的项目安装ng-packagr依赖
```javascript
npm install ng-packagr --save-dev
```
#### 2.在根目录下创建文件ng-package.json,public-api.ts并修改package.json
##### 创建ng-package.json
```javascript
{
    "$schema": "./node_modules/ng-packagr/ng-package.schema.json",
    "whitelistedNonPeerDependencies": [
        "."
    ],
    "lib": {
        "entryFile": "public-api.ts"
    }
}
```
##### 创建public-api.ts
```javascript
/**
 * 导出module模块
 */
export * from './src/app/wxq-module/wxq-module.module'
/**
 * 导出普通类
 */
export * from './src/app/class/user'
/**如果有其他需要导出的对象都可以写在public-api.ts里**/
```
##### 修改package.json：增加打包命令package,并且修改private为false,调整version
```javascript
{
  "name": "yaya",
  "version": "0.0.0",
  "license": "MIT",
  "scripts": {
    "ng": "ng",
    "start": "ng serve",
    "build": "ng build --prod",
    "test": "ng test",
    "lint": "ng lint",
    "e2e": "ng e2e",
    "package": "ng-packagr -p ng-package.json"
  },
  "private": false
   ...
}
```
#### 3.在根目录下执行通过指令生成dist目录，该目录就是我们要发布到npm的内容
```javascript
npm run package
```
#### 4.发布
```javascript
## 设置成本地仓库
// npm config set http://localhost:4873
 nrm add wxqnpm http://localhost:4873
 nrm use wxqnpm

## 登陆npm,(没有先add)
npm login
Username: wxq
Password:
Email: (this IS public) im_luxqi@163.com
Logged in as wxq on http://localhost:4873

# 登陆完就可以publish
cd dist
npm publish
```
## 三、使用
#### 1.install安装
```javascript
npm install yaya-npm-publish-demo@0.0.0 --save
```
#### 2.普通类可通过import命令导入
```javascript
import{Data} from 'yaya-npm-publish-demo';
```
#### 3.组件和正常组件使用方法一样
```javascript
<app-yaya-component></app-yaya-component>
```
## 四、好用的NPM registry管理工具nrm
nrm 可以方便的新增，查看，使用registry,使用安装都起来非常方便
#### 1.安装
```javascript
$ npm install -g nrm
```
#### 2.常规炒作
```javascript
$ nrm add wxqnpm http://localhost:4873 # 添加本地的npm镜像地址
$ nrm use wxqnpm # 使用本址的镜像地址
$ nrm --help  # 查看nrm命令帮助
$ nrm ls # 列出可用的 npm 镜像地址
```