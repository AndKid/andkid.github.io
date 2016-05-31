---
title: 自动编译服务Travis CI介绍
date: 2016-05-30 16:00:01
tags:
- build
- github
---
[Write-Blog-As-A-Geek](https://andkid.github.io/2015/10/09/write-blog-as-a-geek/)这篇文章已经介绍了如何利用Hexo工具生成基于markdown的静态博客网站  
如果嫌弃要装太多工具，推荐一个免费在线IDE  
https://c9.io/  
![Cloud9](/uploads/cloud9.png)  
支持sudo shell，但是出于安全考虑，只支持https协议访问，并不支持http协议  
所以在hexo博客搭好后，需要deploy到github验证  
我注册的时候最后一步图片验证码显示不出来，翻墙后才正常，不确定是不是网络差的关系，总之，这儿顺便推荐个翻墙工具  
https://github.com/getlantern/lantern  
我试了它的apk版本，连接速度很快  
好了，在搞定hexo后，我们写完博文后需要怎么操作呢？
## 传统模式
![Hexo Traditional Workflow](/uploads/hexo_traditional_workflow.png)  
传统模式把编辑和转换放在本地执行，然后把静态的html同步到github  
从图中可以看出最大的问题是如果异地编辑，想要立马部署到github上需要公司和家都要搭建hexo环境才行  
这怎么能忍受的了！  

## 经典模式
![Hexo Classic Workflow](/uploads/hexo_classic_workflow.png)  
经典模式的工作流程：  
1. 在本地将修改后的markdown文档与github的blog分支进行同步
2. github接收到提交后通知编译服务器Travis CI
3. Travis CI得到消息，立马同步blog分支的修改并安排编译
4. Travis CI将编译好的结果同步到github的master分支
5. github自动部署成github page，可通过域名xx.github.io进行访问

这个流程好处是，在任何地点，只要能和github进行同步，就能随时编辑markdown文档，并能实时部署  
狂跩炫酷叼炸天！

## 手把手记录实现步骤
