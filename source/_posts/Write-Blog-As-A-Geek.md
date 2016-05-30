---
title: Write Blog As A Geek
date: 2015-10-09 22:02:58
tags:
- hexo
- git
- blog
---
如何构建类似本站的博客框架，主要有三个部分组成
* github
* hexo
* markdown

# GitHub
## 使用GitHub
访问并注册[github](https://github.com)

创建一个[repo](https://help.github.com/articles/create-a-repo/)，命名为`xxxx.github.io`，这么命名是为了到时可直接当成域名访问

## 关联到本地
下载并安装[Git](https://git-scm.com/)

打开Git Bash，利用[SSH](https://help.github.com/categories/ssh/)关联github网站，这样每次本地修改需要同步到服务器就很方便，不用每次都输密码
需要注意的是，git clone的时候用的是`git@github.com:AndKid/andkid.github.io.git`，而不是https打头的地址

# Hexo
## 安装
[Hexo帮助文档](http://wiki.jikexueyuan.com/project/hexo-document/)
先安装[Node.js](https://nodejs.org/)，也可从我的[网盘](http://pan.baidu.com/s/1i4gai4t)下载
安装完成后执行命令

    npm install -g hexo-cli

## 基本使用
```

hexo init xx  初始化一个网站目录
cd xx
npm install  安装package.json指定的插件
hexo new xxx   创建博文
hexo generate 生成静态网站，可简写成hexo g
hexo deploy   部署到服务器，可简写成hexo d
```
部署成功后就能通过`xxxx.github.io`访问了

在不执行`hexo deploy`部署到远程服务器的情况下，也是可以通过执行本地建站

    hexo server
通过localhost:4000进行本地访问

## 文件结构
```
    .
    ├── _config.yml
    ├── package.json
    ├── scaffolds
    ├── source
    |   ├── _posts
    └── themes
```
`_config.yml`是网站的总配置文件，网站名，作者，远程服务器，主题等
themes是存放主题相关资源的地方，其也有配置文件，也可修改部分源码或资源达到修改目的
source存放md文件，也就是博文内容
## 其他使用
更换[主题](http://hexo.io/themes/)

node_modules下是package.json指定的一些插件，如`hexo-deployer-git`
修改lib/deploy.js

    + var sourceDir = this.source_dir;
    ...
    - return fs.copyDir(publicDir, deployDir)
    + return (fs.copyDir(publicDir, deployDir) && fs.copyDir(sourceDir, deployDir));
可将source目录下的md文档一同部署到服务器，以便异地编辑

# Markdown
[markdown](https://guides.github.com/features/mastering-markdown/)
支持html语言
