---
title: Mov2pic Server Introduction
date: 2016-04-25 23:08:58
tags:
- server
- python
- flask
---
>讲个故事说明本server成因。某一个晚上思念佳人，彻夜难眠，于是乎我打开手机上某学院视频教程寻找催眠良药，
再于是乎看到了python爬虫教程，之前听过网络爬虫这个词，但是理解都是模模糊糊的，反正肯定不是指网虫。。
结果大半夜的，花了两个小时把整套教程看完了，一句话概括，就是获取网站HTML文件显示出来包含文字图片视频等数据。
接下来两天，就写了个爬虫程序把这个某学院视频教程的付费网站的视频给打包爬到本地了，师夷长技以制夷。。
再后来，就想着写个Android App嘛，愁于没有资源，就想到盗取..`读书人窃书不算偷的`，就想到借用人家的资源。
好了，之后就借用了[图解电影网](http://k165.com)和[All in one](http://www.wufafuwu.com/a/ONE_tupian/)文字和图片资源。
源码在[mov2pic_server](https://github.com/AndKid/mov2pic_server),资源以json格式返回。

## 部署
[这篇文档](http://www.jianshu.com/p/be9dd421fb8d)写得很详细了，除了不需要nginx，其他都一样  
我用的windows电脑，也试过用Django部署，但是一些库支持不好，解决了n个错误后还是放弃了  
我再把部署flask用到的命令再贴一下吧
首先需要有个linux的终端应用，[GIT](https://git-scm.com/)这个可以
```bash
mkdir myproject
cd myproject
virtualenv venv #隔离python环境，防止python版本不兼容
source venv/bin/activate #激活隔离环境
pip install flask #安装flask
pip install gunicorn #gunicorn用作flask的容器，简单配置一下就好了
pip freeze > requirements.txt #会把之前通过pip install的都写到requerements.txt文件中，方便移植pip install -r requirements.txt就能都装上了
pip install supervisor #用于进程管理
echo_supervisord_conf > supervisor.conf #生成 supervisor 默认配置文件
supervisord -c supervisor.conf                             #通过配置文件启动supervisor
supervisorctl -c supervisor.conf status                    #察看supervisor的状态
supervisorctl -c supervisor.conf reload                    #重新载入 配置文件
supervisorctl -c supervisor.conf start [all]|[appname]     #启动指定/所有 supervisor管理的程序进程
supervisorctl -c supervisor.conf stop [all]|[appname]      #关闭指定/所有 supervisor管理的程序进程
```
具体看文档吧

## 实现
```
|--hello.py
|--executor
|--|--net.py //获取html内容
|--|--parse_index.py
|--|--parse_content.py
|--|--parse_common.py //以上三个解析图解电影网资源
|--|--parserme.py
|--|--parse_foreground.py
|--|--parse_background.py //以上三个解析All in one资源

```
主要的python库是用于获取html内容的requests库和用于解析html的lxml库  
其中lxml库安装可能会遇到点麻烦  
安装lxml时会报一些错误：
error: Unable to find vcvarsall.bat（用如下2解决）  
当pip install lxml报其他错，试试用easy_install  
	1. 安装python2.7
	2. 安装 Microsoft Visual C++ Compiler for Python 2.7（http://www.microsoft.com/en-us/download/details.aspx?id=44266）
	3. easy_install lxml
ps:可能需要升级setuptools才能自动找到vc compiler的安装路径  
http://stackoverflow.com/questions/26140192/microsoft-visual-c-compiler-for-python-2-7
