---
title: '机器学习之一：基本概念'
layout: post
toc: true
mathjax: true
categories: 技术
tags:
    - Machine Learning
---

# 基本术语

数据记录的集合称为“数据集”（data set），其中每条记录是关于一个事件或对象的描述，称为一个“示例”（instance）或“样本”（sample）。反映事件或对象在某方面的表现或性质的事项，称为“属性”（attribute）或“特征”（feature）。属性上的取值，称为“属性值”（attribute value）。属性张成的空间称为“属性空间”（attribute space）、“样本空间”（sample space）或“输入空间”。

一般地，令 $ D = {x_1, x_2, ..., x_m} $ 表示包含 m 个示例的数据集，每个示例由 d 个属性描述，则每个示例 $ x_i = (x_{i1}; x_{i2}; ... ; x_{id}) $ 是 d 维样本空间 $ X $ 中的一个向量，$ x_i \in X $，其中 $ x_{ij} $ 是 $ x_i $ 在第 j 个属性上的取值，d 称为样本 $ x_i $ 的“维数”（dimensionality）。

从数据中学得模型的过程称为“学习”（learning）或“训练”（training），这个过程通过执行某个学习算法来成。训练过程中使用的数据称为“训练数据”（training data），其中每个样本称为一个“训练样本”（training sample），训练样本组成的集合称为“训练集”（training set）。学得模型对应了关于数据的某种潜在的规律，因此亦称为“假设”（hypothesis）；这种潜在规律自身，则称为“真想”或“真实”（ground-truth），学习过程就是为了找出或逼近真相。

要建立关于“预测”（prediction）的模型，我们需要获得训练样本的“结果”信息，称为“标记”（label）；拥有了标记信息的示例，则称为“样例”（example）。一般地，用 $ (x_i, y_i) $ 表示第 i 个样例，其中 $ y_i \in Y $ 是示例 $ x_i $ 的标记， $ Y $ 是所有标记的集合，亦称为“标记空间”（label space）或“输出空间”。

若预测的是离散值，此类学习任务称为“分类”（classification）；若预测的是连续值，此类学习任务称为“回归”（regression）。对只涉及两个类别的“二分类”（binary classification）任务，通常称其中一个类为“正类”（positive class），另一个类为“反类”（negative class）；涉及多个类别时，则称为“多分类”（multi-class classification）任务。一般地，预测任务是希望通过对训练集 $ {(x_1, y_1), (x_2, y_2), ..., (x_m, y_m)} $ 进行学习，建立一个从输入空间 $ X $ 到输出空间 $ Y $ 的映射 $ f: X -> Y $ 。

学得模型后，使用其进行预测的过程称为“测试”（testing），被预测的样本称为“测试样本”（testing sample）。

还可以对训练集中的数据做“聚类”（clustering），即将训练集中的数据分成若干组，每组称为一个“簇”（cluster）。

根据训练数据是否拥有标记信息，学习任务可大致划分为两大类：“监督学习”（supervised learning）和“无监督学习”（unsupervised learning），分类和回归是前者的代表，而聚类则是后者的代表。

# 假设空间

归纳（induction）与演绎（deduction）是科学推理的两大基本手段。前者是从特殊到一半的“泛化”（generalization）过程，即从具体的事实归结出一般性规律；后者则是从一般到特殊的“特化”（specialization）过程，即从基础原理推演出具体状况。

归纳亦称为“归纳学习”（inductive learging）。归纳学习有狭义与广义之分，广义的归纳学习大体相当于从样例中学习，而狭义的归纳学习则要求从训练数据中学得概念（concept），因此亦称为“概念学习”或“概念形成”。

概念学习中最基本的是布尔概念学习，即对“是”“不是”这样的可表示为0/1布尔值的目标概念的学习。