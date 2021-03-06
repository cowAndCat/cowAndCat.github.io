---
layout: post
title: 机器学习（深度学习）为什么需要训练？
category: ML
comments: false
---

# 机器学习（深度学习）为什么需要训练，训练出来的模型具体又是什么？

 很多新手在初学机器学习/深度学习中，会产生这样的疑问？为什么要训练模型，模型是什么，如何训练......


## 1、机器学习中大概有如下步骤： 

- 确定模型----训练模型----使用模型。
- 模型简单说可以理解为函数。
- 确定模型是说自己认为这些数据的特征符合哪个函数。
- 训练模型就是用已有的数据，通过一些方法（最优化或者其他方法）确定函数的参数，参数确定后的函数就是训练的结果，使用模型就是把新的数据代入函数求值。

通俗来讲：

　　你可以把机器想象成一个小孩子，你带小孩去公园。公园里有很多人在遛狗。    
　　简单起见，咱们先考虑二元分类问题。你告诉小孩这个动物是狗，那个也是狗。但突然一只猫跑过来，你告诉他，这个不是狗。久而久之，小孩就会产生认知模式。这个学习过程，就叫“training”。所形成的认知模式，就是”model“。  
　　训练之后。这时，再跑过来一个动物时，你问小孩，这个是狗吧？他会回答，是/否。这个就叫，predict。
　　一个模型中，有很多参数。有些参数，可以通过训练获得，比如logistic模型中的权重。但有些参数，通过训练无法获得，被称为”Hyperparameter (超参数)“，比如学习率等。这需要靠经验，过着grid search的方法去寻找。  
　　上面这个例子，是有人告诉小孩，样本的正确分类，这叫监督学习。  
　　还有无督管学习，比如小孩自发性对动物的相似性进行辨识和分类。

## 2、如何训练模型？

首先得定义一个损失函数，加入输入样本，根据前向传播得到预测试。跟真实样本比较，得到损失值，接着采用反向传播，更新权值（参数），来回不断地迭代，直到损失函数很小，准确率达到理想值即可。这时的参数就是模型需要的参数。即构建了理想的模型。