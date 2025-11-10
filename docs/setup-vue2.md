---
draft: true
comments: true
date: 2025-05-12
tags: [vue,frontend]
---

# 初始化vue2
推荐方法3中的`vue create my-project`

## method1
手动创建各种文件

mkdir vue2-app
cd vue2-app

npm init -y
npm install vue@legacy
npm install vite @vitejs/plugin-vue2 vue-template-compiler -D


创建文件vite.config.js， 内容：
```
// vite.config.js
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue2'

export default defineConfig({
  plugins: [vue()]
})
```
还有其他文件，这种方式显然不是大家通常想要的

## method2 vue-cli 只适用vue2

npm install -g @vue/cli-init
vue init webpack my-project
cd my-project
npm install
npm run dev

## method3 vue 使用vue2和3,所以要手动选择

npm install -g @vue/cli
vue create my-project

然后选择 Manually select features

回车后再选择 Choose Vue version

最后选择 select 2.x when prompted for the version

### method3 可以兼容method2
method3也可以用vue init, 但需要另外安装

npm install -g @vue/cli-init
# vue init now works exactly the same as vue-cli@2.x
vue init webpack my-project


## 对比 method3 中的 `vue create` 和 `vue init` 的vue2结构区别

最大的区别是启动脚本， 前者是
```sh
# cat package.json
{                       
  "name": "my-project",               
  "version": "0.1.0",               
  "private": true,                    
  "scripts": {                    
    "serve": "vue-cli-service serve", 
    "build": "vue-cli-service build",
    "lint": "vue-cli-service lint"                                  
  },   
```
后者
```sh
cat package.json
{                                       
  "name": "webpack-vue2",                
  "version": "1.0.0",         
  "description": "A Vue.js project",                                                            
  "author": "enerygreek <greek_soon@hotmail.com>",
  "private": true,                    
  "scripts": {                                                                                  
    "dev": "webpack-dev-server --inline --progress --config build/webpack.dev.conf.js",
    "start": "npm run dev",
    "unit": "jest --config test/unit/jest.conf.js --coverage",
    "e2e": "node test/e2e/runner.js",
    "test": "npm run unit && npm run e2e",
    "lint": "eslint --ext .js,.vue src test/unit test/e2e/specs",
    "build": "node build/build.js"
  },
```


## 参考
https://cli.vuejs.org/guide/creating-a-project.html#pulling-2-x-templates-legacy


