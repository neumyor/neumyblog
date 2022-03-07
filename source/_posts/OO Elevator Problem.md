---
title: 面向对象设计与构造课程笔记节选
subtitle: 当时这篇博客在系里还算蛮热门的hhh
---

# OO 电梯算法设计与多线程模拟

[TOC]

## 锁和同步块的分析

本次单元作业中，线程间同步是主要的难点。在进行架构设计时，我重点考虑了**通过精简共享对象层次，来简化锁和同步块的设置逻辑**。

同步块中的处理语句，通常与访问、操作某一对象有关。在这里我们也考虑一个问题：**行为互斥的定义**，即我们究竟应该在什么情况下占有对象并构成互斥访问。显然，**电梯问题中的请求传递**可以看做一个Producer-Consumer问题，而**调度器对电梯状态的查看**是一个Read&Write问题。

在第一次作业中，我没有采用调度器，而是直接通过InputThread向ElevatorThread写入请求。在这个过程中只存在简单的Producer-Consumer关系，故我在电梯的请求队列上加锁，实现InputThread放入请求和电梯处理并移除请求的互斥。

在第二次作业中，我使用调度器Controller来缓存InputThread发送的请求，并查看各个电梯的状态并据此分析请求的派发行为。因此这里又涉及一个Producer-Consumer问题，故我将Controller进行了加锁从而规范读写行为。

我刚开始将电梯的状态ElevatorStatus进行了加锁，而将Elevator的请求队列ElevatorBuffer和ElevatorStatus分开看待。随后，我仔细分析了电梯的行为，**发现InputThread在获取ElevatorStatus后就会放入请求，而Elevator处理了请求后就会改变ElevatorStatus**，这意味着ElevatorBuffer和ElevatorStatus具有逻辑上的耦合性，因此我将两者合并为电梯的数据集合——Elevator对象，并将Elevator的行为抽象为策略集合——ElevatorStrategy对象。ElevatorStrategy和Elevator组装为ElevatorRunnable对象，并**针对Elevator这一个数据对象进行加锁**。

在第三次作业中，我**撤销了调度器的设计**，因此不再对Controller进行加锁。同时引入了全局状态对象GlobalEnv，用于**保存各个进程的状态**，从而实现各个线程状态间的感知。Elevator间从等待队列中竞争请求，因此**演变为Read&Write问题**，允许同步读、读写异步。故设置了读写互斥锁等对象（存在在GlobalEnv中）。

## 交互架构设计

### 第一次作业

简单的从InputThread向ElevatorThread写入请求，构成Producer-Consumer模型。

### 第二次作业

添加调度器Controller，接受InputThread发送的请求并缓存，分析每个电梯的状态并决定请求的分发。电梯只面向自己被分派到的请求进行策略调度。因此将整体策略分为两个部分——分派策略ControllerStrategy和调度策略ElevatorStrategy。

### 第三次作业

第三次作业的更新主要在以下几点：

- 将PersonRequest添加换乘策略PersonStrategy，包装为Person对象。
- 利用GlobalEnv对象进行全局对象的保存，并据此调度各个进程的退出。
- 移除ControllerThread，而仅保留Controller数据对象，作为请求的缓存区。
- ElevatorThread从Controller中竞争请求，在每一层先将能够处理的请求预装载至自己的缓存区，然后再开门装入。

Person类的包装是实现换乘的主要保证。通过**Person对PersonRequest的封装**，将换乘的策略写在Person类自身中，同电梯的调度策略分离，降低了电梯的职责负担。

每个Elevator都会同时观察Controller和其他电梯，当InputThread退出后且Controller为空且其他Elevator缓存区没有请求，将退出。Controller在InputThread退出且所有电梯均退出后将退出。

Elevator使用LOOK算法，**具有自主选择乘客的策略，并能够通过预装载来实现电梯间的竞争**，故Controller的分配策略从主动分配改为被动给予，因此Controller不再具有行为，不再需要单独的线程。

## 第三作业架构设计及可拓展性论述
### 量化视角

利用DesigniteJava对代码进行分析，获得了代码的量化评估结果：

<img src="https://img2020.cnblogs.com/blog/2028226/202104/2028226-20210422195559435-63556865.png" alt="image-20210422112735757" style="zoom:80%;" />

![image](https://img2020.cnblogs.com/blog/2028226/202104/2028226-20210422195504112-471816275.png)

从结果上看，代码的整体的**代码行数分布较为均匀且合理**，符合编写规范。核心类的**内聚合度高，复用性高**，符合高内聚低耦合的设计要求。
### 全局变量GlobalEnv

全局变量GlobalEnv应用了单例模式，用于解决共享变量传递的问题，同时能够反映整体程序的全局状态，用于指导各个线程的退出。

<img src="https://img2020.cnblogs.com/blog/2028226/202104/2028226-20210422195535804-304909211.png" alt="image-20210422112735757" style="zoom:80%;" />


### 电梯Elevator

#### 线程运行逻辑

电梯主要依靠ElevatorRunnable执行运行，其主要行为为做出决定，然后执行。

Elevator类除了Status外，还存放了当前电梯的**类型、可抵达楼层等信息**，其由ElevatorFactory进行初始化，由此实现了对电梯类型方面的可拓展性。

Decision包含一个对当前Elevator对象的引用，使得其execute方法能够改变Elevator的状态（修改当前楼层、读写缓存区请求等）。同时，**Decision还能从全局变量GlobalEnv中获取Controller对象，从而在换乘时将请求放回等待队列**。

```java
@Override
    public void run() {
        AnalyzeAndDecide strategy = strategyMap.get(globalenv.getPattern());
        while (true) {
            Decision decision;
            synchronized (elevator) {
                decision = strategy.decide(elevator);
                if (decision == null) {
                    return; // 如果策略决定要退出, 会返回null
                }
            }
            decision.execute(); // 调用execute方法执行决定
        }
    }
```

<img src="https://img2020.cnblogs.com/blog/2028226/202104/2028226-20210422193013164-1380896757.png" alt="image-20210422113112247" style="zoom:80%;" />

![image](https://img2020.cnblogs.com/blog/2028226/202104/2028226-20210422193037410-74740467.png)

#### 电梯调度算法

采用LOOK算法，利用有限状态机进行构造。预装载（preload）即定义了**电梯的竞争行为**，其从调度器中按一定条件竞争请求并预装载到自己的缓存区中。

我并没有针对各种模式进行优化，原因如下：

1. 经过测试，在全随机的Night和Morning情景下，LOOK算法能够比定制化算法具有基本相近的性能，甚至可以更好。
2. 在边界条件下，LOOK算法具有较高的性能，与定制化算法差距不大，不会引发TLE。
3. 考虑互测中可能遇到特意针对定制化策略的边界数据，使用定制化算法可能被狙击。

![image](https://img2020.cnblogs.com/blog/2028226/202104/2028226-20210422193052116-509484603.png)

### PersonRequest的包装——Person类

Person类包含一个对PersonRequest的引用，并根据其分析了当前请求的换乘类型，并利用状态转换实现了换乘状态的转化。在调用了Person的getOut()方法后，其内部状态会改变，从而影响wantToGetIn()和wantToGetOut()的返回值。**由于采用了状态的转换与换乘类型的独立设置**，因此具有较强的拓展性，可以根据不同的类型设置不同的换乘策略而不影响其他类型的换乘行为。

<img src="https://img2020.cnblogs.com/blog/2028226/202104/2028226-20210422193119769-870604675.png" alt="image-20210422112249370" style="zoom:80%;" />

### 线程间交互

线程间**对于共享对象的互斥访问是经过特地论证的**，因此避免了死锁。（事实上我实现了锁的无嵌套，将在接下来谈一下实现的Trick）。

请注意，Controller和GlobalEnv只是一个共享数据对象而并不是一个线程。

具体交互请看下图：

![image](https://img2020.cnblogs.com/blog/2028226/202104/2028226-20210422202557927-263827552.png)

## 实现Deadlock Free的小Trick

在设计架构时我就考虑了过多的复杂的锁会造成锁的嵌套关系复杂，最终导致死锁发生时分析困难。因此，我尽量将访问上具有联系的对象打包成一个对象或共用一个锁，避免锁的嵌套。这也源自我将**数据和行为**分离的设想，这样使得数据之间更为集中，更利于打包成一个对象。

同时，注意**及时释放锁**。这里的及时释放锁既包括在不再访问对象时释放锁，也包括**合理安排行为顺序，从而尽可能早地结束对对象的访问**。这一点上我使用的trick是，在访问某个对象A时，**如果在获得A的锁后需要访问另一个对象B（并获得B的锁）后还需要访问对象A的数据时，先拷贝对象A然后释放A的锁，再访问对象B**。

最终我在程序中实现了没有锁的嵌套情况，这也使得我不用考虑死锁会带来影响。

## Bug测试策略

本单元主要测试方向在于：

1. 死锁：如何寻找对象的死锁情况？如何命中？
2. 低效策略：如何利用边界数据卡出TLE？

对于第一点，我采用**直接分析代码**的方式。当然，分析也需要策略。编写程序通过对synchronize关键字的嵌套查找，来自动寻找各个对象的嵌套关系。**一旦发现在两个不同线程中两个对象的锁可能构成死锁**，则尽可能设计数据诱导死锁的发生。诱导过程主要通过**高数据量的针对性数据**来实现。

对于第二点，采用特殊数据，以边界条件去测试调度算法。特殊数据的构造纯粹出于人为分析题目要求并构造，此处不再赘述。

## 心得体会

### 线程安全

线程安全方面，我认为**前期分析和架构设计非常重要**。尽可能早地精简共享对象，使得共享关系更加简洁，避免锁的嵌套或设计合适策略来处理死锁（比如利用信号量）。只要在架构上论证了线程间的安全性，那么就可以大胆去进行具体代码的编写，无需在编写过程中再去考虑线程安全问题，这样能够提高效率也能降低后期Debug的难度。

### 层次化设计

我理解的层次化设计，是**将复杂的业务逻辑通过分析，拆分成多个层次的简单逻辑**。在本次实验中**将复杂的策略分担给多个对象，每个对象实现较为简单的某个特定方向上的策略**，极大地降低了de策略bug时的难度，也使得我更加便捷地测试不同策略组合对于性能的影响。