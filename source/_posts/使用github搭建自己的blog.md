---
title: 使用github搭建自己的blog
abbrlink: b7d807be
date: 2020-02-16 11:19:02
tags: hexo
---
　　关于搭建blog的想法已经有很长时间了，虽然现在已经有成熟的框架能够快速搭建自己的blog服务，但是由于自己执行力的原因，一直没有实施。<!--more-->由于这次疫情原因在家休假时间比较长，终于完成了这一项任务。现将自己的操作过程记录一下。
## 技术选型
　　针对类似blog这种服务，现在有很多现成的免费服务可用，比如传统的博客园、CSDN提供的博客服务，以及最近几年出现的简书、知乎专栏、掘金等应用，都能满足大多数人记录文字的需求。或许是出于性格原因，我更想拥有一个自己的域名，自己完全能够控制的站点，所以排除了在以上提到的服务中选择的可能性。
　　刚开始考虑过购买VPS服务来实现，但是想到我现阶段的使用目的只是单纯记录我在后续工作、生活中的一些事情，也不追求访问速度，[Github Pages](https://help.github.com/cn/github/working-with-github-pages)服务可以满足我的需求（其中一个主要原因是Github Pages可以**免费**提供静态站点服务）。
在确定好服务端之后，接着就是考虑内容生成工具，在网上查了一圈之后,从所需要准备的环境，以及内容编写工具以及维护程度等方面来考虑，最后还是选择**Hexo**。
## 搭建过程
　　由于自己的电脑上已经有nodejs、git等hexo所需要的工具，所以在搭建的过程中很快速，步骤也很简单（但是自己还是折腾了好长时间，后面再述），首先使用命令`npm install -g hexo-cli`安装好hexo基础环境，可执行程序。关于hexo教程可以到[这里](https://hexo.io/zh-cn/docs/)学习。在基础环境齐全之后，操作如下命令：
- `hexo init "project name"`，创建一个Hexo项目，后续的操作都在这个项目文件夹里面进行；
- `hexo new [layout] "title"`，创建一个文档，后续编写blog内容都可以通过这个命令来创建对应的md文件;默认情况下，创建的文档位于`source/_posts/`路径下；
- `hexo generate` 将该项目下的文档生成静态网页，生成内容位于`public`目录下；
- `hexo server` 启动一个本地服务，默认访问地址为`http://localhost:4000/`，可以通过该地址的内容进行预览；
通过上面几步，一个blog的内容生成，预览基本上就完成了，但是我们还需要将生成的内容上传到github仓库中。通过Github Pages介绍我们知道，要想启用这个服务，我们需要在自己的github仓库中创建一个 "user".github.io的项目，然后将public中的内容上传到该项目中即可。
为方便上传内容到github，需要安装插件`npm install hexo-deployer-git`，同时在打开根目录中的_config.yml文件，在文档最后部分`# Deployment`增加如下配置，
```yml
deploy:
  type: git
  repo: git@github.com:jeremymj/jeremymj.github.io.git
  branch: master
```
其中repo为将要上传的路径，为减少上传代码时需要输入github用户名、密码，我在这里使用了ssh方式上传，使用https方式也是可以的。在完成上述配置后，就可以使用命令`hexo deploy`将代码上传到github对应仓库中。
## 自定义域名
　　在通过以上几个步骤的操作，正常情况下在浏览器中输入地址项目地址，比如我的:jeremymj.github.io,就可以打开blog的首页。但是假如我们不想使用这个项目地址访问blog，则我们可以为我们的blog绑定一个自己的域名。关于域名的购买，自己随便选择一个服务商就行。在域名中增加一条CNAME解析，eg：`CNAME	www.jspace.top	jeremymj.github.io`,不同的DNS服务商这里显示根式有所区别。正常情况下，一段时间后就可以通过配置的域名访问我们的blog了。
### 支持https访问
　　现在github pages服务直接支持https服务，默认情况下使用`xxx.github.io`的方式访问，直接是https方式。针对自定义的域名路径，支持https也不再需要通过中间代理就可以直实现。操作方式也很方便，在我们创建的github.io项目仓库中找到`Settings`选项，找到`GitHub Pages`设置模块，在`Custom domain`中填上我们自定义的域名，选中`Enforce HTTPS ` 选项即可。为了是刚才的配置生效，我们还需要到为我们的域名在家几条`A`类型记录，**主机记录**这个字段，在有些服务商的配置文件中是填写`@`,有些服务商直接不支持`@`类型的则这个字段不用填写，保持默认即可。对应的值为`185.199.108.153`、`185.199.109.153`、`185.199.110.153`、`185.199.111.153`中的任意一个，这里我增加了4条`A`类型记录。配置好后，正常情况下，一段时间后就可以通过我们自定义的域名使用https访问我们的blog。
## 主题更换
　　选择hexo做为blog的内容生成工具，是因为它有很多[主题模板](https://hexo.io/themes/)供我们使用。我在尝试了几个主题后，考虑到github提供的服务在国内访问速度的问题，最后还是使用的一个大家用的比较多的next模板，主题的替换很简单，在项目的根目录下使用命令:`git clone https://github.com/theme-next/hexo-theme-next themes/next`,将主题模板取回来放置在themes文件夹下，如果要启用该模板，修改项目文件夹下_config.yml文件**theme**字段，值设置为：next。剩下的就是修改next文件下的配置文件，启用该主题的各种样式效果。
## 代码管理
　　为后续代码管理以及编写文档的保存，需要把项目上传到github管理，这里存在一个发布后的文件和源文件。根据我们在[搭建过程](#搭建过程)中对_config.yml的设置，我们把发布后的文件保存在项目的master分支，则针对源文件就上传到该项目下一个分支来保存。在项目根目录下使用命令`git init`将该项目转换为git管理项目，添加远程仓库地址`git remote add origin git@github.com:jeremymj/jeremymj.github.io.git`，创建`.gitignore`文件，选择哪些文件不需要上传到github来管理。接着我们开始创建分支gh_dev，使用命令`git branch gh_dev`,使用命令`git push origin gh_dev`则可以把源文件提交到远程分支来管理。**以后的操作都在本地分支gh_dev来完成,在编辑完成后通过hexo deploy发布到远端mater即可**
## 遇到的问题
　　这里列一些在搭建的过程中遇到的问题:
### cb() never called
　　在使用npm 方式来安装hexo 可执行程序时，遇到速度很慢的问题，但是在我的环境里面启用了自动代理，照理说应该会走代理的，就使用命令`npm config set proxy=http://127.0.0.1:1080`配置代理，后面拉取代码的速度果然有明显的提升，但是在拉取代码的过程中，出现错误提示，`cb() never called`。最后进入项目目录尝试使用`npm install` 获取刚才没有拉下来的代码，也是出现错误提示，在一篇帖子的提醒下，想起在配置代理的时候，当时查看的那个网页上给出的教程是 
```bash
npm config set proxy=http://127.0.0.1:8087
npm config set registry=http://registry.npmjs.org
```
但是我想到只需要proxy,就没有配置registry,在添加好registry后，再次使用`npm install`，**cb() never called！**没有再次出现了，对于出现这个问题的原因，猜测是在默认情况下，npm在执行过程中会有默认registry。我在添加代理的过程中只指定了proxy,系统不再使用默认的registry，对于这个猜测，感觉也不合理，没有道理在用户没有指定registry时，不使用默认registry！由于现在没有对应环境来验证，先在这里记录一下。
### 自定义域名失效
　　在测试的过程中，发现在使用`hexo deploy`后，有时会存在自定义域名不生效，需要在Settings界面重新设置。最后在源文件目录(本地gh_dev分支)中增加一个CNAME配置文件，在配置文件中填写`www.jspace.top`即可以解决这个问题
## 总结
　　以上就是使用hexo搭建blog的过程，关于使用next主题配置的操作，也不再详细说，里面的配置项很多，针对配置项不能够满足我们对效果期望的地方，可以直接修改对应的css文件。关于样式的美化，需要花费的时间比较多，后续再慢慢的优化，至少目前是有一个地方，可以记录自己的学习笔记。