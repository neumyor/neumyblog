---
title: 对结对编程的个人理解
subtitle: 开飞机，芜湖，起飞
date: 2022-03-07 10:00:00
mathjax: true
---

# 结对编程 Pair Programming

## How to define it？

> Pair programming consists of two programmers sharing a single workstation (one screen, keyboard and mouse among the pair). The programmer at the keyboard is usually called the “driver”, the other, also actively involved in the programming task but focusing more on overall direction is the “navigator”; it is expected that the programmers swap roles every few minutes or so.

我们常说的**Agile** Software Development，可以理解为一系列方法以及实践的总称。这些方法各不相同，但是都围绕着敏捷的价值观。其中，我们又可以specify出**XP**，也就是极限编程框架。他是敏捷开发方法中的一个典型代表。而就在XP的核心价值观下，我们又可以进一步定义出Pair Programming。

从形式上来说，Pair Programming可以简单地被描述为在找一个伙伴，一人编码实践一人审查并提出建议，两人保持有规律的轮换以避免疲劳。这一过程是在同一台机器、同一个工位完成的。

## Why do we need it?

XP中的核心价值观可以被描述为：**沟通、简单、反馈、勇气、尊重**，而我们可以从这些角度去思考Pair Programming的意义：

- 沟通：结对编程意味着高强度的交流互动，是**持续的、无间断的、口头的交流**。XP的概念中，这种交流是协作的基础，脱离这种交流将带来极大的风险。从实践上看，这种交流方式在效率上远高于文档和报表。这种沟通同时也能更好地**促进团队内的知识传播**，使得每个组件都能够为更多的开发人员所了解。
- 反馈：XP强调团队和用户之间的反馈，也强调团队内部的反馈。对于后者，Pair Programming显然能够在两位Programmer之间提供**及时的持续的明确的反馈**。XP的概念中，这种反馈能够及时暴露软件状态中存在的问题，显著**提高代码质量**。
- 勇气：XP中勇气指的是Programmer需要拥有面对无法预知的快速需求变化时的勇气。Pair Programming在枯燥的开发过程中带来了一位对等的伙伴，这使得Programmer从心理上更加勇于应对挑战。

（我不大好去定义Pair Programming在简单和尊重方面的意义，因为感觉作用比较有限）

除此之外，Pair Programming还带了一些其他显著优势：

- 将协调的工作直接减半。协调管理$N$个成员，总是比协调$N/2$个成员简单。
- 降低被打断带来的影响。当编程过程被外界因素打断时（外卖到了/工作群里有新消息），一个人可以去处理这些事情，而另一个人可以继续专心编程。

## One more thing……

Pair Programming 也有适用场景的限制，如果出现以下情境需要谨慎使用：

- 处于探索阶段的项目，需要大量脑力思考：写算法、打ACM等
- 后期维护工作且技术含量不高：此时保持复审制度即可，没必要浪费一个人力守在旁边
- 两位Programmer素来关系不和
- 两位Programmer有明显等级差距：比如“老带新”，新人很可能非常被动，使得沟通过程很地狱

## One last thing……

我有看到一些关于Pair Programming实践中，如何使用合适的语言和方式来推进两人的合作。包括如何说服对方、如何读懂潜台词、如何观测对方的肢体语言……我觉得吧，这实在有点太detail了【笑】，有兴趣的话可以看看[两人如何合作做汉堡包](https://www.cnblogs.com/xinz/archive/2011/08/22/2148776.html)