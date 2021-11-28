---
title:  "Minimal Mistake搭建记录"
search: true
categories:
  - 网站搭建
tags:
  - GitHub Pages
  - Minimal Mistake
toc: true
---

## Github Pages

GitHub可以基于静态网页的生成博客网站。在自己的账号下，新建一个`name.github.io`的新仓库就可以启用该功能。`name`要和自己的GitHub ID一致，访问`name.github.io`打开静态博客页面。

## Jekyll和Minimal Mistake

GitHub Pages的功能基于Jekyll实现，默认的主题网页包含信息太少，这里使用人气很高的Minimal Mistake主题搭建我的GitHub Pages博客。

### 本地配置Minimal Mistake

clone Github博客仓库到本地后，前往Minimal Mistake的Github仓库，下载最新的release，解压到本地文件夹。参照Miimal Mistake的文档很容易在本地把静态页面博客跑起来。这样本地自定义的博客页面修改，可以实时调试反馈，比上传到Github仓库查看页面样式方便很多。

### Minimal Mistake的配置

主要的配置选项都在_config.yml文件中。在这个文件中可以设置博客文件目录`_posts`，自定义页面目录`_pages`。修改`_data/navigation.yml`文件自定义网页的头部导航栏。

## 写博客与发布

博客内容全部基于Markdown格式的文档，按照`yyyy-mm-dd-name.md`的格式新建Markdown文件，Jekyll可以从文件名获取博客写作日期。在文档开头添加下面的信息，可以为博文添加分类信息和关键字tags。

```markdown
---
title:  "Minimal Mistake搭建记录"
search: true
<!-- 分类，无需提前定义 -->
categories:
  - 网站搭建
<!-- 关键字标签，无需提前定义 -->
tags:
  - GitHub Pages
  - Minimal Mistake
<!-- 生成目录 -->
toc: true
---
```
