---
title: 使用Hexo搭建个人博客
date: 2019-01-07 21:27:51
categories: "个人学习"
tags: [Hexo,个人博客]
---

<Excerpt in index | 首页摘要> 

# 前言

​	2018年终总结就提到要搭建一个个人博客，没想到这么快就完成了，其实使用Hexo+Github搭建博客还是挺方便的，这里记录下搭建过程。

<!-- more -->
<The rest of contents | 余下全文>

[我的博客](https://dailingnan.github.io/)

(域名还在备案，暂且先通过github地址访问)

# 搭建

## Hexo

Hexo是高效的静态站点生成框架，基于Node.js。通过 Hexo 你可以轻松地使用 Markdown 编写文章

## 安装步骤

1. 安装Git
2. 安装Node.js
3. 安装Hexo
4. 放到github

## 安装Git

Git其实工作中一直在使用，这里就不演示安装过程了，读者可自行下载安装，Window基本都是next  next就完事了。

成功安装后，输入 `git version`,可以显示git版本信息

![](https://note.youdao.com/yws/api/personal/file/261C08221EC7401DA72CBDBBBEDB2EA0?method=download&shareKey=f66c72ab431994cc1bd75ca8eb0310fd)

这里要注意下，Git安装完成后，要配置下全局邮箱和全局用户名

```java
git config --global user.name "dailingnan"
git config --global user.email "250660705@qq.com"
```



## 安装Node.js

下载地址：<http://nodejs.cn/download>

安装过程同样不在详细列出了，都是next next完事

安装完成可以通过以来命令测试

```
node -v
npm -v
```

npm是安装Node.js的时候就一块安装来的

![](https://note.youdao.com/yws/api/personal/file/3B8D156C54114B99BC41C20F38861ADC?method=download&shareKey=31f7153d8a039b543515e8020c9b9ab3)

如果安装没报错，但是不能执行两个命令出现 commod not found一般就是环境变量在作怪，配置下环境变量就好了

## 安装Hexo

安装Hexo利用安装好的npm就可以了

安装前，先设置npm镜像为淘宝镜像

```java
npm config set registry https://registry.npm.taobao.org
```



```java
npm install hexo-cli -g
```

安装过程中windwos安装可能会出现两个警告，不过不影响安装，是linux的警告

![](https://note.youdao.com/yws/api/personal/file/951AF5F7757448928203BA155AE88A9E?method=download&shareKey=51ee4e394cd6e7d67321696ab097af5c)

显示图上的信息就是安装成功了

补充：（由于博主已经安装过一次了，不敢卸载重装，到csdn找了几个图，文章末尾会将参考地址贴出来）

接下来输入`hexo -v`即可测试是否安装成功，出现版本信息就是安装成功了

![](https://note.youdao.com/yws/api/personal/file/6C3E52FF3BB2429ABFF7628BD8E9746A?method=download&shareKey=1e3fde0da0af2e1b95cf17bdb19f873c)

如果出现commod not found也不必担心没安装好，同样是环境变量在作怪，一般npm安装，就自己配置好了环境变量，不过博主安装没有，自己去配的。

此时，可以先到node.js安装目录，通过cmd 运行 `hexo -v`来检验，没配环境变量，在这里也是可以执行这个命令的

比如博主的地址：D:\kaifa\nodejs\node_global

接下来我们把这个地址配置到path变量下面就好了，此时执行`hexo -v`就能正确的出现版本信息了

接下来找一个自己常用的位置执行Hexo的初始化(右键 git bash)

```java
hexo init hexo 
npm install
hexo g
```

初始化完成后，能看到生成了一个hexo的文件夹，里面生成了hexo的一些文件

![](https://note.youdao.com/yws/api/personal/file/427CB0D79BEB4DD6B959F7AC32D434FD?method=download&shareKey=f218e62434e0a1b9d4d522458cbab7e1)

此时执行`hexo s`启动hexo服务来看一下效果

![](https://note.youdao.com/yws/api/personal/file/BF1EB65DD3014D44A4238E2B494256C6?method=download&shareKey=4229746a2a3421ae1d68f5e31d214b04)

接下来访问`loaclhost:4000`，就能看到我们自己的博客了

![](https://note.youdao.com/yws/api/personal/file/DCE7E6EB0FEA49E6BE4840D2ACE87461?method=download&shareKey=f209c63acbdf639ec9ec3c68cbdce73c)

## 放到github

 ![](https://note.youdao.com/yws/api/personal/file/92B9462062CD4A428417630A6A231629?method=download&shareKey=120d13df3ef38ed6748f24312e47d300)

新建一个Repository来存放hexo的页面，这里没啥特别的，就是名字一定要注意，必须为**用户名.github.io**，这样等会才能访问（gitpage访问方式）

接下来配置下hexo目录表的站点配置文件**_config.yml**

```java
deploy:
  type: git
  repo: git@github.com:dailingnan/dailingnan.github.io.git    
  branch: master
```

上面repo的地址就是仓库的地址

![](https://note.youdao.com/yws/api/personal/file/408DED5C9FBE48B384F67CDD424600EA?method=download&shareKey=aba2897cb933616cee6d208e05f36b4f)

接下来我们先确保本地能先连接github，这里不在讲述怎么连接了，读者可自行尝试

连接成功后，输入`$ ssh -T git@github.com`

出现HI   用户名就可以了

![](https://note.youdao.com/yws/api/personal/file/71FB7CE9F1D94EFA801903174254F1D9?method=download&shareKey=1a6ce525a2122e45ec56fe88690294ed)

接下来就是把我们本地hexo生成的静态页面放到Github上面了

执行`hexo d -g`

![](https://note.youdao.com/yws/api/personal/file/C243B757A8304E909F1DED2941F1307F?method=download&shareKey=3e6b101cb1500d9d5468407b5707c0f4)

接下来就可以通过之前配置仓库时候的 **用户名.github.io**的地址来访我们我们自己的博客了

![](https://note.youdao.com/yws/api/personal/file/EC28824615464EA9BFF7CA4A846422D8?method=download&shareKey=bb67de5764ce4087f5e36f3e5562250a)



补充：博主这里是后面修改了hexo的主题，以及增加了一些小功能插件，读者感兴趣的话可以交流哈

后面读者还可以去购买一个域名，绑定自己的博客，之后就能通过自己的域名访问博客了。读者就购买了一个阿里云的域名，现在还在备案中

# 总结

尝试后才发现搭建自己的个人博客还是挺简单的，希望之后能坚持写博客。博客功能也将在后期完善，目前先把csdn的博客迁移过去。