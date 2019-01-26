---
title: markdown插入本地图片小技巧
date: 2019-01-13 11:27:07
categories: "学习记录"
tags: [markdown,技巧]
---

<Excerpt in index | 首页摘要>

# markdown插入本地图片小技巧



## 背景

​	markdown作为一种普通文本编辑器编写的标记语言，通过简单的标记语法，它可以使普通文本内容具有一定的格式，非常的好用，但是插入本地图片很不方便，接下来我们介绍一种非常实用的上传本地图片的方法。

<!-- more -->
<The rest of contents | 余下全文>

## 插入本地图片的几种方法

以下几种方法参考他人博客，末尾回帖出原文地址

### 插入本地图片

只需要在基础语法的括号中填入图片的位置路径即可，支持绝对路径和相对路径。
 例如：

- !\[avatar\]\(本地文件路径\)

> 不灵活不好分享，本地图片的路径更改或丢失都会造成markdown文件调不出图。

### 插入网络图片

只需要在基础语法的括号中填入图片的网络链接即可，现在已经有很多免费/收费图床和方便传图的小工具可选。
 例如：

- !\[avatar\]\(网络图片路径\)

> 将图片存在网络服务器上，非常依赖网络。

### 把图片存入markdown文件

用base64转码工具把图片转成一段字符串，然后把字符串填到基础格式中链接的那个位置。

 例如：

!\[avatar\]\[base64str\]

### 使用选型

不得不说，上面几种方法，各有优缺点，但是为了方便以及图片能够正常的显示，还是使用网络图片比较靠谱，所以这里介绍一个通过有道云笔记来保存图片，生成网络地址的方法。



## 使用有道云生成图片网络地址

表示博主也刚使用markdown不久，之前就被这个本地图片困扰，用一种方式，在csdn会出现图片不能打开，要重新上传假设此时我们有一个本地图片才能访问，后面使用Hexo搭建个人博客后，这种方式完全行不通了，所以找了一种通过有道云生成图片网络地址方法（图床工具没找到）。

假设我们需要添加到markdown文件中，我们只需要把图片保存到有道云笔记中

![](https://note.youdao.com/yws/api/personal/file/ED521C4E092049C3A481712DCEC8971B?method=download&shareKey=d6b1c5c831266cd95b3e6aed5ef69122)

然后右键分享图片

![](https://note.youdao.com/yws/api/personal/file/ECC50E2305FA4C3A830C6684E7C94921?method=download&shareKey=89c53daf983fb567d097006ae957abb7)

查看分享

![](https://note.youdao.com/yws/api/personal/file/96597DCF8C9F4037896828D8B0F76227?method=download&shareKey=3bcfe28b7bd4652ede33f00b931c3e76)

右键复制图片地址

```java
https://note.youdao.com/yws/api/personal/file/66294A284E5D4290BC9524515C60F9C9?method=download&shareKey=2ac22444598f8c3eda2c87705881b0ff
```

将（）中的地址改成网络地址就可以通过网络来访问我们上传的本地图片啦，不过这个图片不能删除哦，删除了网络地址也将不能正常访问了

![avatar](/home/picture/1.png)

## 总结

使用有道云把我们需要用到的图片保存起来，通过图片地址就可以实现图片的不限制访问了，是不是很方便。

参考链接
[简书](https://www.jianshu.com/p/280c6a6f2594)