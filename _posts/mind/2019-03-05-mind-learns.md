---
layout: post
title: 如何改变固有思路
category: mind
comments: true
---

## 一、如何改变固有思路

标题暴露了我目前的问题：思路窄，想法少。

个人觉得原因有很多，这里就不自爆。不过自己从没有放弃对该问题做正向改变。

## 二、经验教训

### 2.1 Docker镜像过大导致加载失败（2019-03-05）

问题背景：同事打包了一个超大的镜像（14.6G），我负责部署校验。结果在load的时候出错，而且每次出错点都在加载导3G大小的时候。另外大部分时间都耗费在文件的拷贝上。

问题解决办法：

- 反例1：尝试高达5次，然后还自行build镜像，切换到其他环境。  
  结果：问题依旧存在。

- 反例2：尝试识别后麻烦两个同事帮忙解决，结果并没有解决问题，还耗费了沟通时间和同事的时间。

最终问题解决方法：作者修改Dockerfile，不将数据打包到镜像里，而是通过文件挂载的方式来实现。

效果：1. 减少了大文件的拷贝； 2. 解决了最终的load问题。

反思：

1. 遇到错误的时候可以重试，但是不要超过3次。如果在重试方法上没有丝毫改进，那么后续操作纯属浪费时间，而且会因为失败产生挫败感。此时应该跳出重试和环境的圈，思考一下镜像的制作方式。
2. 如果可能，尽量不要麻烦其他的同事。