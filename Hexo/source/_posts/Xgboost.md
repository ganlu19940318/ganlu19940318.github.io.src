---
title: XGBoost
date: 2019-6-11 22:24:15
categories: 技术储备
tags: [机器学习]
mathjax: true
---

----

<!-- more -->

# 1. 前言

本文主要参考[了解xgboost么，请详细说说它的原理](https://www.julyedu.com/question/big/kp_id/23/ques_id/2590)
备份:[了解xgboost么，请详细说说它的原理](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/%E4%BA%86%E8%A7%A3xgboost%E4%B9%88.mhtml)

# 2. 前置概念(决策树)

决策树有两种主要类型:

1. 分类树分析是指预测结果是数据所属的类(比如某个电影去看还是不看)
2. 回归树分析是指预测结果可以被认为是实数(例如房屋的价格,或患者在医院中的逗留时间)

术语分类和回归树(CART,Classification And Regression Tree)分析是用于指代上述两种树的总称.

## 2.1 分类树

问题: 如何判断哪个分类效果更好.

思路:需要一个评价标准来量化分类效果.

解决方案: 量化分类效果的方式有很多,比如信息增益(ID3),信息增益率(C4.5),基尼系数(CART)等等.

分类树的灵魂:即依靠某种指标进行树的分裂达到分类的目的,总是希望纯度越高越好.

### 2.1.1 信息增益的度量标准:熵

ID3算法的核心思想就是以信息增益度量属性选择,选择分裂后信息增益最大的属性进行分裂.

什么是信息增益呢?为了精确地定义信息增益,我们先定义信息论中广泛使用的一个度量标准,称为熵,它刻画了任意样例集的纯度.给定包含关于某个目标概念的正反样例的样例集S,那么S相对这个布尔型分类的熵为:

$$
Entropy\left( S \right) =-p_+\log _2p_+-p_-\log _2p_-
$$

上述公式中,$p_+$代表正样例,而$p_-$则代表反样例.(在有关熵的所有计算中我们定义$\text{0*}\log 0=0$)

如果S的所有成员属于同一类,则Entropy(S)=0;
如果S的正反样例数量相等,则Entropy(S)=1;
如果S的正反样例数量不等,则熵介于0,1之间.

通过Entropy的值,就能评估当前分类树的分类效果好坏了.

## 2.2 回归树

背景: 分类与回归是两个很接近的问题,分类的目标是根据已知样本的某些特征,判断一个新的样本属于哪种已知的样本类,它的结果是离散值.而回归的结果是连续的值.

问题: 节点不再是类别,是数值(预测值),那么怎么确定呢?

解决方案: 有的是节点内样本均值,有的是最优化算出来的比如Xgboost.

## 2.3 CART回归树

CART回归树是假设树为二叉树,通过不断将特征进行分裂.比如当前树结点是基于第j个特征值进行分裂的,设该特征值小于s的样本划分为左子树,大于s的样本划分为右子树.

![CART回归树](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20190615112819.jpg)

而CART回归树实质上就是在该特征维度对样本空间进行划分,而这种空间划分的优化是一种NP难问题,因此,在决策树模型中是使用启发式方法解决.典型CART回归树产生的目标函数为:

$$
\sum_{x_i\epsilon R_m}{\left( y_i-f\left( x_i \right) \right)}^2
$$

因此,当我们为了求解最优的切分特征j和最优的切分点s,就转化为求解这么一个目标函数:

$$
\underset{j,s}{\min}\left[ \underset{c_1}{\min}\sum_{x_i\epsilon R_1\left( j,s \right)}{\left( y_i-c_1 \right) ^2+\underset{c_2}{\min}\sum_{x_i\epsilon R_2\left( j,s \right)}{\left( y_i-c_2 \right) ^2}} \right]
$$

所以我们只要遍历所有特征的的所有切分点,就能找到最优的切分特征和切分点.最终得到一棵回归树.

# 3. 前置概念(集成学习)

所谓集成学习,是指构建多个分类器(弱分类器)对数据集进行预测,然后用某种策略将多个分类器预测的结果集成起来,作为最终预测结果.它要求每个弱分类器具备一定的"准确性",分类器之间具备"差异性".

集成学习根据各个弱分类器之间有无依赖关系,分为Boosting和Bagging两大流派.

1. Boosting流派,各分类器之间有依赖关系,必须串行,比如AdaBoost,GBDT(Gradient Boosting Decision Tree),Xgboost.
2. Bagging流派,各分类器之间没有依赖关系,可各自并行,比如随机森林(Random Forest)

## 3.1 AdaBoost

AdaBoost(Adaptive Boosting,自适应增强).

基本思想:前一个基本分类器分错的样本会得到加强,加权后的全体样本再次被用来训练下一个基本分类器.同时,在每一轮中加入一个新的弱分类器,直到达到某个预定的足够小的错误率或达到预先指定的最大迭代次数.

迭代步骤:

1. 初始化训练数据的权值分布.如果有N个样本,则每一个训练样本最开始时都被赋予相同的权值:1/N.
2. 训练弱分类器.具体训练过程中,如果某个样本点已经被准确地分类,那么在构造下一个训练集中,它的权值就被降低;相反,如果某个样本点没有被准确地分类,那么它的权值就得到提高.然后,权值更新过的样本集被用于训练下一个分类器,整个训练过程如此迭代地进行下去.
3. 将各个训练得到的弱分类器组合成强分类器.各个弱分类器的训练过程结束后,加大分类误差率小的弱分类器的权重,使其在最终的分类函数中起着较大的决定作用,而降低分类误差率大的弱分类器的权重,使其在最终的分类函数中起着较小的决定作用.换言之,误差率低的弱分类器在最终分类器中占的权重较大,否则较小.

## 3.2 GBDT

GBDT(Gradient Boost Decision Tree)

基本思想:GBDT每一次的计算是都为了减少上一次的残差,进而在残差减少(负梯度)的方向上建立一个新的模型.

## 3.3 Boosting集成学习

Boosting集成学习由多个相关联的决策树联合决策.

什么叫相关联?举个例子

1. 有一个样本[ 数据 -> 标签 ]是:[ (2,4,5) -> 4]
2. 第一棵决策树用这个样本训练的预测为3.3
3. 那么第二棵决策树训练时的输入,这个样本就变成了: [(2，4，5)-> 0.7]
4. 也就是说,下一棵决策树输入样本会与前面决策树的训练和预测相关

一个决策树形成的关键点在于:

1. 分裂点依据什么来划分(如均方误差最小,loss);
2. 分类后的节点预测值是多少(如有一种是将叶子节点下各样本实际值得均值作为叶子节点预测误差,或者计算所得)

# 4. XGBoost

XGBoost本质上还是一个GBDT,但是力争把速度和效率发挥到极致.所以叫X(Extreme)GBoost.

核心算法思想:

1. 不断地添加树,不断地进行特征分裂来生长一棵树,每次添加一个树,其实是学习一个新函数,去拟合上次预测的残差.
2. 当我们训练完成得到k棵树,我们要预测一个样本的分数,其实就是根据这个样本的特征,在每棵树中会落到对应的一个叶子节点,每个叶子节点就对应一个分数.
3. 最后只需要将每棵树对应的分数加起来就是该样本的预测值.

# 5. 参考链接

[了解xgboost么，请详细说说它的原理](https://www.julyedu.com/question/big/kp_id/23/ques_id/2590)
[一文读懂机器学习大杀器XGBoost原理](http://blog.itpub.net/31542119/viewspace-2199549/)