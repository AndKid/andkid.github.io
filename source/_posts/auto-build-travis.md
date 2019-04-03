---
title: 自动编译服务Travis CI介绍
date: 2016-05-30 16:00:01
tags:
- blog
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

## 实现步骤
### 为github绑定Travis CI服务
![Add Travis CI Service](/uploads/add_service.png)  
在your_name.github.io的Settings中Webhooks&services选项，通过Add service添加Travis CI服务  
授权后即可绑定Travis CI，通过 https://travis-ci.org/ 可利用github帐号登录进行配置  
![Travis CI Settings](/uploads/travis_settings.png)  
在项目根目录下创建一个.travis.yml，Travis CI会执行该文件中指定的命令  
示例  
.travis.yml
```yml
language: node_js

node_js:
    - 'stable'

sudo: false

before_install:
- openssl aes-256-cbc -K $encrypted_26b4962af0e7_key -iv $encrypted_26b4962af0e7_iv
  -in .travis/ssh_key.enc -out ~/.ssh/id_rsa -d
- chmod 600 ~/.ssh/id_rsa
- eval $(ssh-agent)
- ssh-add ~/.ssh/id_rsa
- cp .travis/ssh_config ~/.ssh/config
- git config --global user.name 'AndKid'
- git config --global user.email cygcloud@163.com
- git clone -b master git@github.com:AndKid/andkid.github.io.git .deploy_git

install:
- npm install hexo-cli -g
- npm install

script:
- hexo clean
- hexo g
- hexo d

branches:
  only:
  - blog

```
后边会解释每一行具体意义  
### 生成SSH Key
Travis CI在成功将markdown转换成html后需要向github同步，但是它并没有权限  
因为项目是开源的，我们不能直接把帐号密码写在文件中  
所以，我们采取的方法是生成SSH对称密钥，利用数字签名，即私钥加密，公钥解密进行身份验证  
`ssh-keygen -t rsa -C "your_email@example.com"`
> 注意存放路径，不要覆盖Linux默认~/.ssh  

### 将公钥告知github
将public key放在github的blog的git中  
![SSH Public Key](/uploads/ssh_public_key.png)  

### 将私钥告知Travis CI
将private key告知Travis CI  
当然，private key也不能明目张胆地保存在项目中  
同样，我们需要利用SSH Key对这个private key进行加密，这次我们采用公钥加密，私钥解密的方式来传输隐私文件  
这次我们不需要自己生成，因为Travis CI提供了命令工具来做这件事  
`gem install travis`  
gem是ruby的工具，不想安装可利用最上面提到的processon在线IDE  
`travis login --auto`
利用github帐号登录，执行该命令后会告诉你放心登录，不会窃取你的密码～
`travis encrypt-file id_rsa --add`
执行后会自动在.travis.yml插入一行解密命令，Travis CI会执行它，如  
`- openssl aes-256-cbc -K $encrypted_26b4962af0e7_key -iv $encrypted_26b4962af0e7_iv
  -in .travis/ssh_key.enc -out ~/.ssh/id_rsa -d`  
注意这个是执行encrypt-file执行完后自动插入的，不要照抄，-in是encrypt-file加密后的.enc文件，这个文件就可以放心地放在项目中，我放在.travis目录下  

### 配置Travis CI SSH
以上只是将公钥和私钥放置在了合适的位置  
Travis CI在向github提交代码如何利用SSH附带私钥信息呢，需要以下几步  
```yml
- chmod 600 ~/.ssh/id_rsa
- eval $(ssh-agent)
- ssh-add ~/.ssh/id_rsa
- cp .travis/ssh_config ~/.ssh/config
```
ssh_config内容为  
```
Host github.com
  User git
  StrictHostKeyChecking no
  IdentityFile ~/.ssh/id_rsa
  IdentitiesOnly yes
```
至此，Travis CI向github提交时会用私钥加密  

### .travis.yml剩余命令
```yml
- git config --global user.name 'AndKid' #配置git提交时的用户名
- git config --global user.email cygcloud@163.com #配置git提交时的email
- git clone -b master git@github.com:AndKid/andkid.github.io.git .deploy_git #将master分支clone到根目录，用于转换完成后merge并提交

install:
- npm install hexo-cli -g #安装hexo
- npm install #安装hexo插件，在package.json中指定

script:
- hexo clean #清理环境
- hexo g #生成静态html
- hexo d #部署到github

branches:
  only:
  - blog #监听blog分支改动的消息

```
至此，配置已经完成，接下来只要往github的blog分支提交代码  
Travis CI就会把静态博客部署到master分支  
相关配置可参考 https://github.com/AndKid/andkid.github.io/tree/blog  
Enjoy！

参考文档：  
http://www.v2ex.com/t/170462  
https://zespia.tw/blog/2015/01/21/continuous-deployment-to-github-with-travis/
