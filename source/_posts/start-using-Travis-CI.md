---
title: 使用Travis CI进行持续集成
date: 2017-07-25 19:51:19
tags:
 - 工具
 - CI
categories:
 - 工具
---

Travis CI是当前最受欢迎的CI套件，Saas模式的服务独树一帜，流畅的使用方案让人欲罢不能。
作为一个Saas模式的CI服务提供者，Travis CI为你的每次构建提供一个独立的构建环境，支持[多种语言](https://docs.travis-ci.com/user/languages/)，预置了多种环境配置模式，内置了多种发布模式。
本文以Hexo博客构建并发布到github pages,以及Node工程构建并发布到私有服务器为例，介绍Travis的使用方式，帮助大家为自己的项目增加一枚构建徽记。
[![Build Status](https://www.travis-ci.org/BroKun/blogSource.svg?branch=master)](https://www.travis-ci.org/BroKun/blogSource)

<!--more-->

## 在Travis CI上注册并关联项目
对于github上的开源项目，访问[Travis CI](https://www.travis-ci.org/),点击右上角使用github帐号登录。
![www.travis-ci.org](http://blog-1253747550.cossh.myqcloud.com/Images/travis-ci/index.png)

登录以后在右上角的头像下，选择Accounts，会看到如下界面。
![repo](http://blog-1253747550.cossh.myqcloud.com/Images/travis-ci/repoList.png)

travis已经给了你很详细的使用指导：
  1.把开关打开 
  2.添加配置文件 
  3.提交代码触发。

---

  好了，全文完，谢谢大家！

---

大家在这里把开关打开就算完成了。

## Hexo博客构建和部署github pages的配置
Hexo虽然提供了deploy到github的支持，但是对于更换机器等情况而言，依赖越多用起来越复杂，想随时随地往仓库里push一个md文件，就完成发布的话，就要考虑自动构建的方式了。

使用travis来完成这项工作有两个选择:
* 使用travis在自己的deploy选项，travis在近期加入了pages发布支持，只要配置几个主要信息，travis就可以完成发布动作。这个模式现在还是实验状态的，但是亲测可用。
* 使用构建环境，手动把文件提交的github pages仓库。

### 获取github token
两种方式都需要使用github access token。所以我们先获取它。

登录到github→在右上角头像的下拉选项中进入Settings→找到Personal access tokens选项→点击右上角Generate new token→填入描述信息，选好权限→mission complete!

生成的token信息请妥善保管。

### travis环境设置
登录到travis，回到刚才repo列表页面，需要构建的项目开关旁边有一个设置按钮，点击进入设置。

在General中把构建的触发条件选好。
![settings-general](http://blog-1253747550.cossh.myqcloud.com/Images/travis-ci/settings-general.png)

在Environment Variables中添加github token到环境变量
![settings-environment-variables](http://blog-1253747550.cossh.myqcloud.com/Images/travis-ci/settings-environment-variables.png)

### 使用travis的deploy
在项目中添加.travis.yml文件，内容如下:
```
language: node_js
node_js: stable
install:
- npm install
script:
- hexo g
deploy:
  provider: pages
  local_dir: public/ #用于deploy的文件夹
  repo: BroKun/BroKun.github.io #Repo slug
  target_branch: master #分支
  email: brokun0128@gmail.com #提交动作的使用的email
  name: BroKun #提交动作使用的帐号
  skip_cleanup: true
  github_token: ${github_token} # Set in travis-ci.org dashboard
  on:
    branch: master
branches:
  only:
  - master
```
depoly的详细说明参见[官方文档](https://docs.travis-ci.com/user/deployment/pages/)

### 使用手动提交的方式
在项目中添加.travis.yml文件，内容如下:
```
language: node_js
node_js: stable
install:
- npm install
script:
- hexo g
after_script:
- cd ./public
- git init
- git config user.name "BroKun"
- git config user.email "brokun0128@gmail.com"
- git add .
- git commit -m "Update docs"
- git push --force --quiet "https://${github_token}@${github_ref}" master:master
branches:
  only:
  - master
env:
  global:
  - github_ref: github.com/BroKun/BroKun.github.io.git
```

## 构建Node工程并发布到私有服务器
如果希望把工程发布到自己的服务器，需要需要做的会多一点，首先要明确如何去做这件事。我的选择是构建过程和测试在travis的环境中运行，通过ssh把源码发送到远程服务器，在远程执行部署脚本。

### 配置远程ssh登录
由于travis不支持交互式指令，所以采用ssh免密远程登陆的方式来访问远程服务器。
#### 生成密钥对
```shell
$ ssh-keygen -t rsa
```
这里注意，生成私钥的时候，密码设置为空。
#### 将公钥放在远程服务器上并设置受信
```shell
$ cat id_dsa.pub >> ~/.ssh/authorized_keys 
$ chmod 600 ~/.ssh/authorized_keys
```
#### 在需要部署的项目中添加私钥并加密
在项目的git托管目录中，添加.travis文件夹，将刚刚生成的私钥放入文件夹中。熟悉密钥登录方式的应该知道，我们正常是在自己的机器上保存私钥，把公钥给自己管理的服务器，这里我们是希望travis的环境替代我们自己的机器。但是私钥放置在项目中意味着暴露在了网络空间，是很不安全的，所以我们使用[The Travis Client](https://github.com/travis-ci/travis.rb)加密私钥。

安装Travis Client:
```shell
$ gem install travis
```
PS:Travis Client是基于Ruby的，需要安装Ruby环境支持，我的机器为此增加了ruby rubygems ruby-devel等包，如果出现ruby版本问题，建议使用RVM。

登录到Travis
```shell
$ travis login
```
填入自己的travis帐号或github帐号信息

加密私钥
```shell
$ travis encrypt-file id_rsa --add
```
--add参数会在git仓库内的.travis.yml中增加
```
before_install:
- openssl aes-256-cbc -K $encrypted_cf2f27b2e746_key -iv $encrypted_cf2f27b2e746_iv
  -in id_rsa.enc -out id_rsa -d
```
在travis网站上的项目配置里，也可以看到增加了两个环境变量
![加密过的环境变量](http://blog-1253747550.cossh.myqcloud.com/Images/travis-ci/encrypted.png)

travis把解密需要的信息保存在了自己的服务器上，我们就不用担心私钥信息丢失了，这时候把项目仓库.travis文件夹里的原始私钥删除，只保留id_rsa.enc就可以了。

### 在.travis.yml中编写构建和发布过程
一个完整的例子如下:
```
sudo: false
language: node_js
node_js:
- '7.10.1'

before_install:
- openssl aes-256-cbc -K $encrypted_cf2f27b2e746_key -iv $encrypted_cf2f27b2e746_iv
  -in .travis/id_rsa.enc -out ~/.ssh/id_rsa -d
- chmod 600 ~/.ssh/id_rsa
- echo -e "Host 101.200.36.181\n\tStrictHostKeyChecking no\n" >> ~/.ssh/config
- tar -czf relationship-server.tar.gz *
install:
- npm i npminstall && npminstall
script:
- npm run ci
after_script:
- npminstall codecov && codecov
- npm i coveralls
- cat ./coverage/lcov.info | node ./node_modules/coveralls/bin/coveralls
- scp relationship-server.tar.gz brokun@101.200.36.181:~/
- ssh brokun@101.200.36.181 'rm -rf relationship-server'
- ssh brokun@101.200.36.181 'mkdir -p relationship-server && tar -xzvf relationship-server.tar.gz -C relationship-server'
- ssh brokun@101.200.36.181 '. ./relationship-server.sh'
```
这里的Node程序使用了Egg框架，在travis上执行lint与单元测试。结束之后使用[coveralls](https://coveralls.io/)发布了测试结果。然后发布程序，在远程服务器上我预先放置了部署脚本，来完成程序重启相关的工作。
需要注意的有:   
1. Travis Client自动生成的解密脚本不能给出正确的加解密文件路径，需要手动调整。
2. 私钥的文件访问权限一定要修改，脚本里还在.ssh文件夹增加了配置，关闭了对指定服务器的严格检查，否则会出现需要用户确认的步骤，导致脚本无法执行。
