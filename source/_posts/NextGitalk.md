title: Next添加Gitalk评论系统
author: ZhiPeng Wang
date: 2019-11-28 12:52:56
tags: [Next, Gitalk]
categories: Hexo
---
## 介绍

最近在搭建Hexo + Next 的博客系统，在添加评论系统时选用了gitalk系统作为首选的评论系统，但是在添加过程中遇到了很多问题，在经过了好几次测试，终于将gitalk 与 Next 7.5.0 结合在一起。

## 步骤

### 创建存储仓库

首先需要在github上创建一个新的public的仓库，名称为[blog-comment](https://github.com/wozipa/blog-comment)

### 创建Application

创建地址为：[https://github.com/settings/applications/new](https://github.com/settings/applications/new)
![upload successful](/images/pasted-0.png)

注意：Callback URL 填写的是博客的连接


![upload successful](/images/pasted-1.png)
创建完成后可以在Settings/ Developer settings中查看创建好的ClientID和 Client Secret

### 修改Next主题配置文件

修改Next目录下的 _config.yml文件，在其中找到gitalk的选项直接修改：

```
gitalk:
  enable: true
  github_id: wozipa
  owner: wozipa
  repo: blog-comment
  client_id: xxx  # 填写上图的Client ID
  client_secret: xxx # 填写上图的Client Secret
  admin_user: wozipa
  distraction_free_mode: true # Facebook-like distraction free mode
  # Gitalk's display language depends on user's browser or system environment
  # If you want everyone visiting your site to see a uniform language, you can set a force language value
  # Available values: en | es-ES | fr | ru | zh-CN | zh-TW
  language: zh-Hans
```

### 修改gitalk文件

gitalk.swig文件夹在 ```themes/next/layout/_third-party/comments``` 路径中，打开gitalk.swig后，将原有内容全部删除，并添加一下的代码

```javascript
{% if page.comments and theme.gitalk.enable %}
	<link rel="stylesheet" href="https://unpkg.com/gitalk/dist/gitalk.css">
	<script src="https://unpkg.com/gitalk/dist/gitalk.min.js"></script>
	<script src="https://cdn.bootcss.com/blueimp-md5/2.10.0/js/md5.min.js"></script>
    // gitalk添加所需要的配置信息
	<script type="text/javascript">
    		var gitalk = new Gitalk({
		        clientID: '{{ theme.gitalk.client_id }}',
		        clientSecret: '{{ theme.gitalk.client_secret }}',
		        id: md5(location.pathname),
		        repo: '{{ theme.gitalk.repo }}',
		        owner: '{{ theme.gitalk.owner }}',
		        admin: '{{ theme.gitalk.admin_user }}',
			distractionFreeMode: '{{ theme.gitalk.distractionFreeMode }}',

		    });
	    gitalk.render('gitalk-container');
	</script>
{% endif %}
```

### 修改index.swig文件

打开```themes/next/layout/_third-party/index.swig``` 文件并在最后一行添加下面的内容

```
{% include 'comments/gitalk.swig' %}
```

修改完成后，直接hexo clean && hexo g && hexo s ，然后查看效果就可以。


## 问题

### 评论框为出现

当gitalk设置打开的时候，Next会自动为我们生成一个标签，大概就是长下面介个样子：

```
<div id="gitalk-container"></div>
```

具体的截图是：

![upload successful](/images/pasted-2.png)

然后上面的JavaScript代码根据ID值获取这个DIV标签，然后将Gitalk的内容渲染在这个标签中，如果你的评论框未生效时，Check一下图中的位置是否生成了上述这个标签，如果没有生成请在```themes/next/layout/_partials/comments.swig``` 文件进行修改：

```
// 将这里的注释符号去掉
{% if page.comments %}
  
  // 这里的内容不用修改，因此省略掉
  ...
  
  // 添加下面的内容，手动添加git-contianer的DIV标签
   {% if theme.gitalk.enable %}
 	<div id="gitalk-container"></div>
  {% endif %}
// 这里的注释符号删除
{% endif %}
```
然后继续查看页面是否出现了指定的标签。


## 总结

Next 7.5.0添加Gitalk的过程已经分享给大家，如果有什么不正确的地方请大家指出来哈~
