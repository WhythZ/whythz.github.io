---
# author:
title: 如何用Jekyll与Github搭建博客
description: >-
  这条博客写于此站点成功发布之后，由于我对前后端知识的了解比较浅薄，所以本文仅用于记录下自己作为非开发者搭建Jekyll基础环境的过程以及遇到的一些小问题
date: 2024-08-14 02:41:00 +0800
categories: [其它开发, 站点搭建]
tags: [Github, Jekyll, 环境配置]
# pin: true
# media_subpath: '/resources/'
# render_with_liquid: false
math: true
# mermaid: true
# image:
#   path: /resources/2024-08-15-如何用Jekyll与Github搭建博客/ruby-msys2.png
#   lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
#   alt: Responsive rendering of Chirpy theme on multiple devices
---

## 一、安装Ruby环境

>参考Jekyll的[官方文档](https://jekyllcn.com)，其关于不同操作系统的配置指引在[此处](https://jekyllrb.com/docs/installation/)，此篇笔记如无特殊说明，均以Windows端的操作为例

### 1.1 Windows端
- Jekyll是由Ruby编写的，所以要安装配置Ruby环境
- 如果是Windows端的话，从[此处](https://rubyinstaller.org/downloads/)下载安装`WITH DEVKIT`版本进行安装
- 安装完成后还有一个安装步骤如下（若不小心退出了就使用`ridk install`即可重新进入），选择选项3，等待安装完成后不需要进行下一次数字的输入，直接退出即可

![ruby-msys2.png](/resources/2024-08-14-如何用Jekyll与Github搭建博客/ruby-msys2.png)

- 打开`cmd`命令行输入下面的指令检测`Ruby`是否安装成功

```
ruby -v
```

### 1.2 Linux端
- 如果是Linux端的话，先执行以下命令以使用最新的稳定版本更新系统

```
apt update -y  
apt upgrade -y
sudo apt update
```

- 然后使用以下指令安装Ruby

```
sudo apt-get install ruby-full build-essential zlib1g-dev
```

- 安装完成后，我们需要告诉Ruby的gem包管理器将gems放在用户的主文件夹中，我们可以通过修改`~/.bashrc`文件来达成这个效果，通过以下指令打开`~/.bashrc`文件

```
nano ~/.bashrc
```

- 在该文件末尾添加以下内容

```
export GEM_HOME=$HOME/gems
export PATH=$HOME/gems/bin:$PATH
```

- 保存并关闭文件，然后使用以下命令激活环境变量

```
source ~/.bashrc
```

## 二、安装RubyGems
- `RubyGems`是`Ruby`的包管理器，下载链接在[此处](https://rubygems.org/pages/download)，Linux端无需此步骤
- 下载zip文件到本地，然后解压到某目录下，在命令行中输入以下指令

```
cd your_unzipped_rubygem_dir
ruby setup.rb
```

- 等待安装完成后使用下面指令检测

```
gem -v
```

## 三、换源以提速
- 这是针对Windows端的方法

```
gem sources -r https://rubygems.org/ -a https://gems.ruby-china.com/
```

## 四、安装Jekyll
- 在命令行执行此指令以安装

```
gem install jekyll
```

- 安装完成后使用下面的指令检测

```
jekyll -v
```

## 五、Bundler的使用
- 也可以使用`gem install bundle`安装`Bundler`，这样可以更加方便地管理各模板环境的依赖项
- 在Windows端为`Bundler`换源提速

```
bundle config mirror.https://rubygems.org https://mirrors.tuna.tsinghua.edu.cn/rubygems
```

- 安装`Gemfile`中指定的所有依赖的`gem`包

```
bundle install
```

- 更新项目中的`gem`依赖项到最新的兼容版本

```
bundle update
```

- 使用以下命令可以在当前项目依赖的上下文环境中执行命令，比如常用的`bundle exec jekyll server`

```
bundle exec your_command
```

## 六、搭建基于Jekyll的博客
- 在某路径下使用指令创建一个项目，执行完成之后我们就成功创建了一个名为`project_name`的`Jekyll`项目文件夹

```
jekyll new project_name
```

- 我们`cd`进入到该项目目录下执行`jekyll server`或者`jekyll s`命令即可启动本地服务
- 除了自己创建外，我们还可以使用别人的主题模板来搭建，可以在Jekyll主题官网jekyllthemes.org处寻找合适的模板，社区也有海量由开发者制作的主题可以选择
- 如果使用模板的话，一定要依据模板作者的要求，配置好环境，一步一步来即可
- 直到如下图一般成功搭建，我们即可通过其提供的地址在本地实时编辑预览博客了

![jekyll-server运行成功.png](/resources/2024-08-14-如何用Jekyll与Github搭建博客/jekyll-server运行成功.png)

- 在本地调试完毕后，我们即可将模板整合到我们的仓库内并push到Github，然后在对应的仓库设置内创建页面即可，注意大部分主题模板都是以Github Actions的方式创建网页的，而不是像往常一样使用Branch

## 七、遇到的问题
- 执行`jekyll server`命令时如若遇到以下报错，解决方法是执行`gem install minima`，类似的问题解决方法差不多，把缺失的包下载下来即可

```
Could not find gem 'minima (~> 2.5)' in locally installed gems. (Bundler::GemNotFound)
```

- 也有的问题需要通过修改`Gemfile`文件解决，比如我在使用主题模板搭建博客时就遇到过关于`tzinfo`包的报错，仅仅通过`gem install`无法解决报错，需要在`Gemfile`中添加如下所示的`gem "tzinfo-data"`才可以解决

```
# frozen_string_literal: true

source "https://rubygems.org"

gem "jekyll-theme-chirpy", "~> 7.0", ">= 7.0.1"

group :test do
  gem "html-proofer", "~> 5.0"
end

# 若无此项则在Windows端执行jekyll server会产生异常
gem "tzinfo-data"
```