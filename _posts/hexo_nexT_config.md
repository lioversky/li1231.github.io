---
title: hexo使用nexT主题修改参数整理
date: 2018-06-25 15:30:00
categories: 技术
tags: hexo
---

* **主页显示预览**

修改主题配置文件，找到如下内容：

```
# Automatically Excerpt. Not recommend.
# Please use <!-- more --> in the post to control excerpt accurately.
auto_excerpt:
  enable: false
  length: 150
```
enable表示是否开启摘录，默认为false，length表示摘录字数。
<!-- more -->
* **选择 Scheme**

修改主题配置文件，找到如下内容：

```
# Schemes
#scheme: Muse
#scheme: Mist
#scheme: Pisces
scheme: Gemini
```
目前为止提供四种schema可选择使用，只能有一个前面没有#长效，具体区别来自官方文档：
>Muse - 默认 Scheme，这是 NexT 最初的版本，黑白主调，大量留白
>
>Mist - Muse 的紧凑版本，整洁有序的单栏外观

>Pisces - 双栏 Scheme，小家碧玉似的清新


* **语言设置**

修改**站点配置**文件，增加`language: zh-Hans` 配置，更多语言支持见官方文档。

* **菜单设置**

修改主题配置文件，找到如下内容：

```
menu:
  home: / || home
  #about: /about/ || user
  tags: /tags/ || tags
  categories: /categories/ || th
  archives: /archives/ || archive
  #schedule: /schedule/ || calendar
  #sitemap: /sitemap.xml || sitemap
  #commonweal: /404/ || heartbeat
```
将需要保留的菜单前面的#去掉即生效，如果上面已设置为中文，主页显示对应中文。


* **昵称和头像**

修改**站点配置**文件，设置 author 为你的昵称；

修改主题配置文件，找到如下内容：`#avatar: /images/avatar.gif`，将值替换成具体的图片地址，可以是网络地址，也可以是站内相对地址。

* **生成分类页**

1. 执行命令`hexo new page categories`生成分类页面；
2. 修改生成md文件，在文件头部增加`type: "categories"`；
3. 在生成其它博文md文件中增加:`categories: some_ category`
			
* **生成标签页**

1. 执行命令`hexo new page tags`生成标签页面；
2. 修改生成md文件，在文件头部增加`type: "tags"`；
3. 3. 在生成其它博文md文件中增加:
	
``` 
tags:
  - tag1
  - tag2
```

* **社交链接**

修改主题配置文件，找到如下内容：

```
social:
  GitHub: https://github.com/your-user-name
  Twitter: https://twitter.com/your-user-name
  微博: http://weibo.com/your-user-name
  豆瓣: http://douban.com/people/your-user-name
  知乎: http://www.zhihu.com/people/your-user-name
```
将值修改成真实链接即可，没有前面加#。