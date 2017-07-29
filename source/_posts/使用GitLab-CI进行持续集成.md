---
title: 使用GitLab CI进行持续集成
date: 2017-07-23 18:38:48
tags:
 - 工具
 - CI
categories:
 - 工具
---

从GitLab8.0开始，就内置了GitLab CI，不管围绕着gitlab建立了怎样的workflow，CI都是不可缺少的。
如果你的团队使用GitLab，同时又希望研发人员自己承担DevOps的角色，那GitLab CI将是你很好的选择。
<!--more-->

![image](https://docs.gitlab.com/ee/ci/img/cicd_pipeline_infograph.png)

# 特点
* GitLab CI的构建流程依托于每台服务器上的GitLab Runner，Runner需要主动注册到Gitlab CI，免去了复杂认证设置，Runner直接在服务器上运行，对本地的操作也会更方便。
* PipeLine依托在项目中配置的.gitlab-ci.yml文件，构建过程管理与研发流程贴得更近。



## 说明
### GitLab Runner
构建过程的执行者，使用Go语言编写，只要能跑Go的地方就能跑。
Runner分为Specific Runner与Share Runner。Shared Runner只有系统管理元可以创建，为多个项目服务。Specific Runner有工程访问权限的人都可以创建，大多数情况下，Specific Runner足够使用。


# 使用步骤
这里以我需要配置的一个工程为例。
* 服务器操作系统：CentOS 7
* GitLab：8.17.x
* 工程语言: Node.js

## 安装 GitLab Runner
安装GitLab Runner前，首先要搞清楚自己的GitLab版本，根据[官方文档](https://gitlab.com/gitlab-org/gitlab-ci-multi-runner)给出的对应表找到自己可用的Runner版本。我这里选择使用v1.11.x

```
root@host:# yum install gitlab-ci-multi-runner-1.11.2-1
```
GitLab Runner会在服务器上创建gitlab-runner用户，该用户的目录会生成builds文件夹，里面为每个runner建立一个文件夹，文件夹内部放置由该runner管理的信息。

其他参考 [官方说明](https://docs.gitlab.com/runner/install/)

## 注册 GitLab Runner
我这里选择注册一个Specific Runner，需要Url和Token，这两个信息可以从GitLab的项目设置里拿到。位置在设置->runners

![image](/images/gitlab-ci/runner-url-token.png)

```
root@host:# gitlab-runner register
```
然后根据提示依次输入URL、Token等信息   
其中tags信息，Whether to run untagged builds与runner执行的触发有关。   
executor 可以选定命令执行器，为了方便，我选择了shell

注册成功以后，可以看到如下字样：
```
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded! 
```
同时会在 /etc/gitlab-runner/config.toml 生成一份配置文件，可以再次修改部分配置信息，runner会自动重启。
在gitlab的runner配置项中，此时也可以看到该runner的，也可以在这里修改runner描述信息和tags等。
![image](/images/gitlab-ci/runner-added.png)

一个注册好的runner在其他的代码仓库也是可以看到的，可以选择在多个项目中启用。
![image](/images/gitlab-ci/runner-exist.png)

## 配置.gitlab-ci.yml
配置好runner以后，在项目中添加.gitlab-ci.yml文件以触发runner的执行。

### 概念
![image](https://docs.gitlab.com/ee/ci/img/pipelines_grouped.png)

GitLab的CI构建过程分为Pipeline、stage、job三个层次。   
Pipeline是指的一个可执行的构建过程。   
stage是Pipeline的执行阶段。   
job是一个具体的执行单元，job里包含的信息最多，需要指明自己所属的stage，需要执行的命令，在什么情况下执行，在哪里执行等。


### 示例配置文件
先给出配置文件:

```
stages:
  - install
  - test
  - deploy

cache:
  key: ${CI_BUILD_REF_NAME}
  paths:
    - node_modules/

# 安装依赖
install:
  stage: install
  only:
    - dev
  script:
    - npm install

# 运行测试用例
test:
  stage: test
  only:
    - dev
  script:
    - npm run test
  allow_failure: true

# 开发环境部署到pm2
deploy_dev:
  stage: deploy
  only:
    - dev
  tags:
    - dev
  script:
    - pm2 delete dev || true
    - pm2 start config/pm2-dev.json --name dev

# 测试环境部署到pm2
deploy_test:
  stage: deploy
  only:
    - dev
  tags:
    - test
  script:
    - pm2 delete test || true
    - pm2 start config/pm2-test.json --name test
```

这份配置文件描述了一个由install、test、deploy三个stage组成的Pipeline，全部由dev分支的更新触发，install和test两个job是公共的，deploy_dev、deploy_test分别在开发和测试两个环境的runner上执行部署。

### 执行方式和注意事项
gitlab会监视仓库内每个分支上的.gitlab-ci.yml文件，在仓库更新的时候触发构建动作。
一个仓库内可以有多个.gitlab-ci.yml文件，每个文件都可以成为一个Pipeline,GitLab通过对job的解析，job指定的分支与当前分支对应，并且根据tags等信息匹配到runner时，构成一条可用的Pipeline。
可用的Pipeline可以在gitlab上直接查看。
![image](/images/gitlab-ci/pipeline.png)

job通过stage标明自己所属的阶段，通过only和except来选定可以触发执行的分支。通过tags来指定执行命令的具体runner，当没有tags的时候，所有允许执行没有标签的作业的runner来执行命令。
建议大家在配置的时候，建立好tags的对应关系。

### 文件格式
配置文件中除了保留字及其含义，搬运自官方文档:
|Keyword|Required|Description|   
|:------|:-------|:---------|   
|image |no |Use docker image, covered in Use Docker|
|services	|no	|Use docker services, covered in Use Docker|
|stages	|no	|Define build stages|
|types	|no	|Alias for stages (deprecated)|
|before_script	|no	|Define commands that run before each job's script|
|after_script	|no	|Define commands that run after each job's script|
|variables	|no	|Define build variables|
|cache	|no	|Define list of files that should be cached between subsequent runs|

除此之外的key，都被定义为job，在我给出的配置文件中，install、test、deploy_dev、deploy_test均为job。job定义的属性为:
|Keyword	|Required	|Description|  
|:------|:-------|:---------| 
|script	|yes	|Defines a shell script which is executed by Runner|
|image	|no	|Use docker image, covered in Using Docker Images|
|services	|no	|Use docker services, covered in Using Docker Images|
|stage	|no	|Defines a job stage (default: test)|
|type	|no	|Alias for stage|
|variables	|no	|Define job variables on a job level|
|only	|no	|Defines a list of git refs for which job is created|
|except	|no	|Defines a list of git refs for which job is not created|
|tags	|no	|Defines a list of tags which are used to select Runner|
|allow_failure	|no	|Allow job to fail. Failed job doesn't contribute to commit status|
|when	|no	|Define when to run job. Can be on_success, on_failure, always or manual|
|dependencies	|no	|Define other jobs that a job depends on so that you can pass artifacts between them|
|artifacts	|no	|Define list of job artifacts|
|cache	|no	|Define list of files that should be cached between subsequent runs|
|before_script	|no	|Override a set of commands that are executed before job|
|after_script	|no	|Override a set of commands that are executed after job|
|environment	|no	|Defines a name of environment to which deployment is done by this job|
|coverage	|no	|Define code coverage settings for a given job|
|retry	|no	|Define how many times a job can be auto-retried in case of a failure|

更多参考 [官方说明](https://docs.gitlab.com/ee/ci/yaml/README.html)

## 结果查看
配置结束以后，就可以在GitLab的Pipeline和Jobs两个地方看到执行情况了，在Jobs的位置可以看到每个步骤在服务器上的执行情况。
![image](/images/gitlab-ci/jobs.png)
