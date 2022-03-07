---
title: 从Attention 到 MultiHeadAttention
subtitle: 小笔记
---

# 从Attention 到 MultiHeadAttention

## 对Attention 的理解

Attention的本质类似于我们人类的注意力机制，寄希望于**将有限的注意力集中于重点上**，从而节省资源以获得最为有效的信息。

那么对于神经网络来说，什么是注意力呢？而又如何分辨什么是重点？简单来说，Attention机制通过计算知识源中的各个部分与学习目标的相关性，来尽可能学习相关性最高的部分。

> 图书馆（**source**）有很多书（**value**），而为了方便查找，图书馆为每本书进行了编号（**key**），然后给出一个编号区间的表，来映射编号与语义的关系，假设编号1~5为计算机、编号6~10为数学、编号11~15为护理学。当我们需要学习有关人工智能（**query**）的知识时，我们根据这张表去重点阅读1~10号，而对11~15号只进行粗略的浏览。
>
> 这样，我们就能在有限的时间中获得尽可能多的关于人工智能的知识

Attention机制可以被描述为以下步骤：

1. 将query与source中已知的key进行相似度计算，获得每个key的权值

2. 将权值进行归一化，以得到直接可用的权重（$d_k$是$QK^T$方阵的维度）

   > 假设 $Q,K$ 里的元素的均值为0，方差为1，那么$Q K^T$中元素的均值为0，方差为d. 当d变得很大时，$Q K^T$中的元素的方差也会变得很大，如果$QK^T$中的元素方差很大，那么$Softmax(QK^T)$的分布会趋于陡峭(分布的方差大，分布集中在绝对值大的区域)。总结一下就是$Softmax(QK^T)$的分布会和d有关。因此 $Softermax(QK^T)$ 中每一个元素除以$\sqrt d_k$后，方差又变为1。这使得$Softmax(QK^T)$的分布“陡峭”程度与d解耦，从而使得训练过程中梯度值保持稳定。                                                                                                                                                                                                                                                                                                                                                                                                  

3. 将权重和value进行**带权求和**，得到向量Attention value

$$
\text { Attention value }(Q, K, V)=\operatorname{softmax}\left(\frac{Q K^{T}}{\sqrt{d_{k}}}\right) V
$$

对于一段知识源，一次Attention获得了该知识源的一个表示空间，以Attention value的方式表示。

## Self-Attention

然而我们再思考一个问题，对于一段文本，究竟什么是它的query、什么是它的key、什么又是它的value呢？

假设我们的source是这样一段文本“Water is toxic”

当我们处理"Water"这个词时，将"Water“作为当前搜索的query，与其他词对应的key向量进行相似度计算。依次，我们用同样的方法计算剩下的两个单词。

Self-Attention认为(Q, K, V)均来自于source的词向量，即“Water is toxic”这一句话所生成的词向量，因此三者应该在生成机制上是一致的。因此定义了三个相互独立的权重矩阵（作为训练时的需要学习的参数），用于与输入的词向量相乘获得三个矩阵，分别作为Q、K、V。

详细内容查看[超详细图解Self-Attention)](https://zhuanlan.zhihu.com/p/410776234)，非常详细而且易懂。

## Multi-Head Attention

Multi-Head Attention的一个基本思想在于，我们试图通过多个Attention来建立对同一个知识源的多个不同的注意力关系判断。这就好比让多个人来同时思考一个问题，不同的人看待问题的方式不一样，因此让多个人一起能够更好地找到这个问题相关的知识点。

从机制上，其相当于多个Self-Attention过程的融合。假设有n个Head：

1. 每个Head对应三个权重矩阵用于从输入向量中计算(Q, K, V)
2. 每个Head根据自己的(Q, K, V)根据Attention过程计算得出Attention value向量。n个Head一共有n个Attention value。
3. 将这n个Attention value向量连接起来，乘以一个权重矩阵，以使其转变为与输入向量大小相同的矩阵

参考：

- [Multi-Head Attention](https://zhuanlan.zhihu.com/p/266448080)
- [Attention机制详解（二）——Self-Attention与Transformer - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/47282410)
- [一文看懂 Attention（本质原理+3大优点+5大类型） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/91839581)
- [超详细图解Self-Attention - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/410776234)
- [拆 Transformer 系列二：Multi- Head Attention 机制详解 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/109983672)