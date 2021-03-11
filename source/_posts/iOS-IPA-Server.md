---
title: iOS无需越狱在线安装ipa，一键上传ipa，开源代码，自部署应用分发服务，类似fir.im
date: 2017-12-22 20:03:37
tags: ios
---

# 背景
iOS开发经常会需要分发ipa包给相关同事或者朋友对软件进行测试，一般会使用Apple提供的`TestFlight`但是需要额外安装一个app，使用起来不是很方便。实际上苹果提供了一个网页直接安装ipa的途径，国内的一些分发平台就是利用这个特性搭建的，<https://fir.im/>，
 <https://www.pgyer.com/> 都是不错的平台。

# 问题

但是由于一些管制问题，这些平台最近开始都需要实名认证才能上传应用了，`TestFlight`需要额外安装一个app。

# 解决方案

为了方便自己发布应用供朋友安装，于是开发了这个服务
ipa-server: <https://github.com/iineva/ipa-server>。
安装和使用都很简单，里面附带了详细的中文说明。
部署使用的是Docker，对Docker熟悉对童鞋应该两分钟就可以搞定了。觉得好用对话，记得点个赞👍哈

# 快速试用（可以上传和查看，但是不能安装，在iOS设备上进行安装需要HTTPS支持，详细内容看README文档）

```shell
$ git clone https://github.com/iineva/ipa-server
$ cd ipa-server
$ docker-compose up -d
```

# 下面是使用截图

![图片描述][1]
![图片描述][2]


  [1]: https://github.com/iineva/ipa-server/raw/main/snapshot/zh-cn/1.jpeg
  [2]: https://github.com/iineva/ipa-server/raw/main/snapshot/zh-cn/2.jpeg
