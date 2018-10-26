---
layout: post
title: 神经网络
category: ml
comments: false
---
## 一、神经网络的运作过程
一个神经网络网络的搭建，需要三个条件：

1. 输入和输出
2. 权重w和阈值b
3. 多层感知器的结构

其中，最困难的部分就是确定权重（w）和阈值（b）。一般把不断试错来找出合适的w和b的过程叫做模型训练。

因此，NN的运作过程如下：

1. 确定输入和输出
2. 找到一种或多种算法，可以从输入得到输出
3. 找到一组已知答案的数据集，用来训练模型，估算w和b
4. 一旦新的数据产生，输入模型，就可以得到结果，同时对w和b进行校正

识别车牌的例子：

车牌照片就是输入，车牌号码就是输出，照片的清晰度可以设置权重（w）。然后，找到一种或多种图像比对算法，作为感知器。算法的得到结果是一个概率，比如75%的概率可以确定是数字1。这就需要设置一个阈值（b）（比如85%的可信度），低于这个门槛结果就无效。

一组已经识别好的车牌照片，作为训练集数据，输入模型。不断调整各种参数，直至找到正确率最高的参数组合。以后拿到新照片，就可以直接给出结果了。

## 二、今天鱼打完了，明天晒网

## REF
> [神经网络入门](http://www.ruanyifeng.com/blog/2017/07/neural-network.html)
> [Machine Learning Basics](http://www.deeplearningbook.org/contents/ml.html)