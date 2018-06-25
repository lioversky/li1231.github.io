---
title: Docker部署hexo在github搭建博客
date: 2018-06-22 
tags: hexo
categories: 技术
---

# 背景 

在博客网站发布markdown格式的博客越来越方便，但发现好多个人博客的样式大都一样，研究发现好多都是使用github+hexo+个人域名，貌似并不复杂，所以尝试搞起。使用Mac + Docker + Hexo + github，由于不想在自己电脑安装过多的东西，所以docker是最好的选择。
<!-- more -->

# 过程

## 打包hexo镜像

在docker官方hub上找了几个hexo的镜像，发现stars和pull的都不太多，在github上查了一下，对几个做了整合，地址分别为[docker-hexo](https://github.com/yakumioto/docker-hexo) 和[hexo-docker](https://github.com/neoFelhz/hexo-docker)。

最后镜像的Dockerfile如下

```
FROM node:9.5-alpine

RUN apk add --update --no-cache git
RUN npm config set unsafe-perm true 
	&& npm install hexo-cli -g

WORKDIR /Hexo

EXPOSE 4000
```
分别使用过ubuntu和centos做过测试，在启动过程中都出现若干问题，有的是在安装nodejs有的是在执行npm，总之对以上问题没有深入解决使用了最后的方案。

打包镜像完成后，首先初始化，命令如下

```
docker run -it \
	-v /hexo:/Hexo \
	li1231/hexo \
	sh -c 'hexo init . && npm install && npm install hexo-deployer-git --save'
```
将初始化的所有结果都保存到外挂目录内，命令执行完后会在hexo初始化目录结构如下
	
	_config.yml	
	db.json
	node_modules
	package-lock.json
	package.json
	public
	scaffolds
	source
	themes
到此hexo安装成功。

## 本地测试

在source/_posts目录下创建md的文件，也可以使用hexo n创建md文件，使用hexo g命令生成静态页面，使用hexo s生成预览，通过本地访问页面查看效果。因为使用hexo的详细说明特别多，在此不多说。先执行生成命令如下：

```
docker run -it \
	-v /hexo:/Hexo \
	li1231/hexo \
	hexo g
```
再执行预览命令：

```
docker run -it \
	-p 4000:4000 \
	-v /hexo:/Hexo \
	li1231/hexo \
	hexo s
```
开启本机4000端口访问。

## github创建repository

在github自己账号创建 username.github.io的repository，如果有自己域名可以绑定，此教程网上特别详细，此处省略。

## 提交本地博客到github

将本地的静态页面提交到github，要先修改主配置_config.yml，增加

```
deploy:
  type: git
  repo: https://github.com/username/username.github.io.git
  branch: master
```
再执行hexo deploy命令提交

```
docker run -it \
	-v /hexo:/Hexo \
	li1231/hexo \
	hexo d
```
在提交过程中要输入用户名和密码。本以为将本机公钥加到github中，再将.ssh文件夹挂载到docker同时git config --global user.name && git config --global user.email可以直接免密，但目前还没有成功。

参考：

[https://www.cnblogs.com/visugar/p/6821777.html](https://www.cnblogs.com/visugar/p/6821777.html)
