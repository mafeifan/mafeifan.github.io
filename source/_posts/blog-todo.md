---
title: 网站待处理的问题
date: 2017-07-29 17:27:37
tags: website
---


### 更新记录

- 2017.9.01 升级为https
- 2017.8.18 加入 CNZZ 统计
- 2017.8.09 安装 https://github.com/hexojs/hexo-generator-sitemap 插件

<!--more-->

### 已处理问题

- /tags打开报404，没有生成index.html
答：运行`hexo new page tags`, 然后打开`\source\tags\index.md`添加一行`type: tags`
- more查看全文的使用
答：使用`<!--more-->`就可以了
- 如何添加next主题的自定义样式
答：修改`\themes\next\source\css\_custom\custom.styl`
- RSS支持
答：/sitemap.xml



### 网站待处理的问题

- blog版本对比
- 提速，合并JS
- 头像等细节
- 仿 分成技术和生活两大板块

### 在其他电脑上写blog的方法
* git checkout resource分支
* npm install
* hexo g -d 编译并部署
* 注意：永远不要在master分支上手动提交代码，只需在resource分支改动


