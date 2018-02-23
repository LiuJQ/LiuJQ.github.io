---
layout: post
title: 浅谈Jekyll中的相对路径
tags: [Jekyll, 相对路径]
---

&emsp;&emsp;Jekyll是一个伟大的静态网站工具，如果你正在使用GitHub Pages来维护公共博客或个人网站，它将会是一个非常有用的免费工具。但是，使用Jekyll需要注意一个问题——相对路径，该问题暂时还未有一个好的解决方案。

### 问题根源
```html
<!-- _layouts/default.html -->
 <link href='assets/css/style.css' rel='stylesheet'>
```

&emsp;&emsp;假设你的个人博客中某页面需要引用style.css文件，那么就要像上面一样引入。但是assets/css/style.css引用的是相对路径，并不是以/开始的绝对路径，HTML页面解析的时候会从当前目录下找到assets目录，再按照路径去查找style.css文件。

&emsp;&emsp;因此，当你的页面放在项目根目录的时候，上面的引用方式是可以正常显示css样式的。一旦需要引用style.css文件的HTML页面是在更深一层或几层的子目录时，则会发生css文件找不到问题导致页面样式渲染异常。

| 页面路径        | 页面父目录     | 解析路径 | 样式是否渲染 |
| :------------- | :-------------: | :------------- | :-------------: |
| /index.html       | /       | /assets/css/style.css | 是 |
| /about.html       | /       | /assets/css/style.css | 是 |
| /about/profile.html       | /       | /about/assets/css/style.css | 否 |

### 解决方案
&emsp;&emsp;一个简单的解决方案：
```html
 <link href='/assets/css/style.css' rel='stylesheet'>
```
&emsp;&emsp;当你的个人博客工程放在站点主目录 user.github.io/ 时（以Github Pages为例，下同），这个解决方案是可行的，可一旦你尝试把个人博客工程移动到其他目录（比如，user.github.io/project/）时，就得不到你期望的路径 /project/assets/css/style.css 了。

| 博客工程目录 | 解析路径     | 样式是否渲染 |
| :------------- | :------------- | :-------------: |
| user.github.io/       | /assets/css/style.css       | 是 |
| user.github.io/project       | /assets/css/style.css       | 否 |

### 更好的解决方案
&emsp;&emsp;根据页面路径URL动态计算style.css文件的层级并将对应的父目录层级保存在一个JavaScript变量中，在其他需要引用style.css文件的HTML页面中引用这个变量即可。
#### step 1

```html
{% raw %}
<!-- _includes/base.html -->
{% assign base = '' %}
{% assign depth = page.url | split: '/' | size %}
{% if    depth <= 1 %}{% assign base = '.' %}
{% elsif depth == 2 %}{% assign base = '..' %}
{% elsif depth == 3 %}{% assign base = '../..' %}
{% elsif depth == 4 %}{% assign base = '../../..' %}
{% endif %}
{% endraw %}
```

&emsp;&emsp;在_includes文件夹下新建base.html文件，加入上面这段JavaScript代码，然后在你的layout模板HTML文件里面引入_includes/base.html文件。引用方法如下：

```html
{% raw %}
<head>
 ...
 {% include base.html %}
 <link href="{{ base }}/assets/css/style.css" rel='stylesheet'>
 ...
</head>
{% endraw %}
```
#### step 2
&emsp;&emsp;在其他页面中，加入base变量的引用即可。
```html
{% raw %}
 <a href="{{ base }}">Back to home</a>
 <a href="{{ base }}/about.html">About me</a>
 <a href="{{ base }}{{ post.url }}">Read "{{ post.title }}"</a>
 {% endraw %}
```
