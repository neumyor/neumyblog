---
title: C++常用数据结构
subtitle: 不懂C呜呜呜
---

# C++ 常用数据结构及其使用

这是我最近在自学c++过程中，意识到自己对c++的数据结构尚不熟悉，因此搜罗来的各类资料并进行了自己的理解
[TOC]

## array 数组

array在功能上是为了弥补传统C语言数组的越界问题，而由C++提出的更为安全的数组容器。

### array的初始化

数组array类是一个**严格固定长度**的容器，这意味着在声明时就需要声明长度。

```c++
array<int,8> 8byteslongarray;
```

array是一个严格线性的容器，在物理上也是线性排列，可以通过偏移指针来访问元素，访问耗时恒定。

### array的使用

#### 迭代器的使用

array是一个可以通过迭代器iterator进行遍历的容器，可以通过以下方式获取和使用迭代器：

- **begin()**:获取数组开头位置的迭代器

- **end()**

- **rbegin()**：获取数组尾部的迭代器，且反向迭代

- **rend()**

- cbegin()：获取一个const的迭代器对象，**无法通过const iterator去改变一个对象的值！**

- cend()

- crbegin()

- crend()

  ```c++
  for(auto it = arr.rbegin();it!=arr.crend();++it){
  	cout << *it << endl;
  }
  ```

#### 数据的访问

array是严格定长的，允许通过size()、empty()和max_size()进行容量的访问。（我不清楚max_size的作用）。

你可以通过多种方法访问一个array的数据对象

```c++
array[1];
array.at(1);
array.front();
array.back(); 
array.data(); // 获取指向第一个元素的指针
```

注意array是物理上连续的，因此获取指针后可以通过跳转指针地址获取对象（但是出于安全考虑最好不要这样做）

array支持fill来进行内容的填充（不知道初始化时是否会fill全为0），还可以使用swap来交换指向的指针。

```c++
array.fill("wanted value");
arrayA.swap(arrayB);
```

------



## vector 向量

vector是向量类型，是C++中重要的容器对象。其实质上也是采用数组进行实现，需要连续的存储空间，当数据极大时会导致分配内存上的困难。同时vector的优势在于智能管理，更加安全的同时还提供了大量的现有函数，支持自动增长。

### vector的多种初始化方式

vector真是一个功能丰富的类呢（笑）

```c++
vector<int> a(999); // 声明一个999个整数的vector
vector<int> b(666,2); // 声明一个666个整数的vector并全部初始化为2
vector<int> c(b); // 将b整体复制性赋值给c
vector<int> d(c.begin(),c.end()-4);	 // 利用始末迭代器构造
int array[100] = {1,2,3,4,5,6};
int* pointer = array+1;
vector<int> e(pointer,pointer+3); // 利用指针间连续空间进行构造
```

### vector的操作方式

#### vector元素访问与更改

vector支持使用[ ]进行访问，还可以通过front和back获取特殊位置元素。

```c++
v[4]; // 注意下标支持获取值，但是只支持获取size范围内的元素引用，试图获取一个没有声明大小的或者size访问外的元素将会导致错误！
v.at(4); // at稍微安全一点，因为其会返回错误信息指明访问非法
v.front();
v.back();
v.begin(); // 迭代器是最推荐的方法，因为迭代器实现了一个泛化的指针，具有更强的泛化性
v.end(); // 注意这个迭代器没有指向任何有效元素，其实际上指向最后一个元素后面的一个位置
```

vector的元素操作方法极其丰富：

```c++
v.assign(b.begin(),b.end()-3); // 将b的第一个到倒数第4个字符赋值给v
v.assign(4,2); // 给v赋值为4个连续的2
/* 	注意assign会覆盖原有所有数据！*/
v.insert(v.begin()+1,5); // 在v的第二个元素位置插入一个5（使得v[1]==5）
v.insert(v.begin()+1,4,6); // 在v的第二个元素位置插入4个6
v.insert(v.begin()+1,b,b+3); // 在v的第二个元素后插入b数组的前3个元素

v.pop_back(); // 删除最后一个元素（注意这个没有返回值！）
v.push_back(5); // 向v最后插入一个5

v.clear(); // 清除所有元素
v.erase(v.begin(),v.begin()+3); // 删除v的前三个元素
```

#### vector的容量操作

vector是具**有动态拓展性**的，但是也支持用户手动干预其容量。

```c++
a.size(); // 获取vector长度
a.capacity(); // 注意容量总是大于等于长度!

a.empty();

a.resize(100); // 限制长度为100，空余位值随机！
a.resize(100,3); // 限制长度为100，空余位补3
a.reserve(1000); // 手动扩容为1000，以防止系统自动多次扩容带来的性能损耗
```

#### vector的奇技淫巧

vector作为一个极其重要的类，其集成了多种性能较高的算法。

```c++
v.sort(v.begin(),v.end()) // 从小到大排列
v.reverse(v.begin(), v.end()-3) // 翻转
copy(v.begin(),v.end(),b.begin()) // 把v从b的开头开始复制覆盖原有元素
find(a.begin(),a.end(),"target") // 在a中查找特定元素
bool is_odd(T a);
v.erase(remove_if(v.begin(),v.end(),is_odd), array.end()); // 在v中按照is_odd函数删除符合要求的元素
```

------

## deque 双向队列

deque双向队列，支持两端操作。**vector是单向开口的内存空间**，这意味着什么呢？意味着如果你在vector头部插入一个元素，那么会慢得离谱（这个操作是合法的但不是推荐的）。而deque则是更加好的选择，**你可以在头尾很快地进行元素操作**。相应的，其**占用内存会更多**。实际上deque包括两级结构，一级类似于vector，一级则是专门维护首地址。

### deque的初始化

```c++
deque<int> d (a.begin(),a.end());
deque<int> d (a,a+3);
deque<int> d (5,8) // 初始化为5个8
deque<int> d (anotherd); // 拷贝复制
```

### deque操作

#### deque赋值

```c++
d.assign(a.begin(),b.end());
d.assign(5,8);
d.swap(anotherd);
```

#### deque特色

我们说过，deque和vector的区别在于对双端操作的支持更好，而且我们能发现，**deque比vector多了头部操作函数**，除此之外没有特别明显的使用功能区别。(ps：deque还是一个没有容量概念的容器，因此和vector不同的一点在于，vector在空间不够的时候会申请一个新的空间再经过复制和释放旧空间的过程，但是deque会直接将新空间连接到旧空间后。因此deque没有reserve空间的需要)

```c++
d.push_front(element); // 向头部添加一个元素
d.pop_front(element); // 删头部元素
```

------

## list 链表

list是一个顺序容器，其通过链表实现，主要优势在于**快速的插入和删除**。

在功能上，vector、deque、list三者是经常被提及的选择对象，以下列出三者使用场景的不同。

- vector：大量随机访问需求 && 插入删除多在尾部

- list：少量访问需求 && 大量插入和删除需求

- deque：大量访问需求 && 大量插入和删除需求 && 对内存要求不高

### list的初始化

list的初始化与vector类似，在此按下不表

### list的操作

list的操作与vector略有不同，因为其链表的性质，**合并和拆分的功能较为突出**。另外也注意到list的**有序性要求**。注意list没有随机迭代访问器，因此其sort函数是自己定义的。

```c++
list.sort(); // 默认升序排序
list.sort(greater<int>()); // 改为降序排序

l1.merge(l2); // merge要求两个链表都是有序的，默认升序
l1.merge(l2,greater<int>()); // 改为降序排序
/* merge后l1为合并结果，l2为空 */

l1.splice(it1,l2); // 将l2全部剪切并接到it1位置后面，l2为空，it1指向的字符不变，it1-l1.begin()增加了
l1.splice(it1,l2,it2); // 将l2的it2位置字符剪切并接到l1的it1位置（使其成为it1指向对象）
l1.splice(it1,l2,l2.begin(),l2.begin()+2); // 将l2前三个字符剪切并接到l1的it1后
```



------

## map 表

map是我们所讲到的第一个关联容器，是一个对“映射”这种逻辑关系的支持，提供一对一的存储。map这种数据结构**有两种实现**：其一是**基于红黑树的map**，其二是**基于hash表的unorder_map**。前者的数据有序性更好，红黑树会对数据进行自动排序，但是占用空间大；后者对查找的支持更好，但是对于复杂操作的时间效率不够好。这两种的**操作完全一样，只是底层不同**！

### map的数据操作

map有最主要的操作是“插入”，即插入键值对，具有三种插入方式：

```c++
map.insert(pair<T,T>(a,b));
map.insert(map<T,T>::value_type(a,b));
// insert不会覆盖原来的值
map[a]=b;
// 采用下标会覆盖原有的值
```

map的数据查找**比较反人类！**，主要的方法为find

```c++
iter = map.find(a);
if(iter!=map.end()) find it!
// find返回的是对应位置的迭代器，没找到就会返回end()，不对应任何元素
```

map的数据删除有两种方式：

```c++
int flag = map.erase(a); // 删除成功返回1，否则返回0
map.erase(map.begin(),map.end()); // 清空map
```



------

## set 对

set是我们谈到的第二个关联容器，可以看出，关联容器不支持顺序容器的关于位置的操作（因为实际上没有位置的概念）。set中每个元素只是一个关键字，而与map中的键值对区别开来。

set的关联容器包括两大类四种：

- set：采用高效的平衡检索二叉树：红黑树

  - unordered_set：基于hash函数实现的set

- multiset：支持关键字重复出现的set

  - unordered_multiset

set的**主要作用在于查询**，即查询在该结构中存在特定关键字。set的元素会**默认升序排列**（默认比较函数为less\<int\>）

### set的初始化

```c++
int * a = {12,3,4,5};
array<int> b {1,2,3};
set<int> s (a,a+3); // 使用数组初始化
set<int> s (b.begin(),b.end()); // 使用迭代器初始化，很好理解，因为迭代器就是更加泛化的指针
set<int> s {1,2,3,4}; // 使用初始化列表初始化
```



### set的使用

首先，set支持一般的迭代器，因此你可以利用迭代器的连续变化来获取set内的值（这里体现了迭代器的泛化性，因为set实际上不是顺序存储的，但是利用起迭代器来和顺序容器完全一样）

```c++
for (it=s.begin();it!=s.end();it++){
	cout << *it << endl;
}
```

其次，我们说过set的主要作用在于寻找元素是否存在，因此传统艺能不能丢（指count和find）

```c++
s.count("target"); // 返回0或者1,0表示不存在
s.find("target"); // 返回迭代器，不存在则返回s.end()
```

set的元素增删采用insert和erase实现

```c++
s.erase("target");
s.erase(s.begin(),s.being()+3);
s.insert("me");
s.insert(vector.begin(),vector.end());
// erase和insert支持传入迭代器区间操作
```



------

## stack 栈

### 如何初始化一个stack？

stack 容器适配器的模板有两个参数。**第一个参数是存储对象的类型**，**第二个参数是底层容器的类型**。stack<T> 的底层容器默认是 deque<T> 容器，因此模板类型其实是 stack<typename T, typename Container=deque<T>>。通过指定第二个模板类型参数，可以使用任意类型的底层容器，**只要它们支持 back()、push_back()、pop_back()、empty()、size() 这些操作**。下面展示了如何定义一个使用 list<T> 的堆栈：

```
std::stack<T,std::list<T>> stackOnList;
```

初始化一个堆栈时，**不能在初始化列表{}中用对象来初始化**，但是可以用另一个容器来初始化，只要**堆栈的底层容器类型和这个容器的类型相同**。

```c++
std::list<T> l {1,2,3};
std::stack<int, std::list<int>> mystack(l);
```

不过stack**支持拷贝构造**，这意味着在初始化列表使用一个stack能够制造一个副本：

```c++
std::stack<int, std::list<int>> anotherstack {mystack};
```

### stack类的操作

stack 是一类存储机制简单、所提供操作较少的容器。下面是 stack 容器可以提供的一套完整操作：

- **top()**：返回一个栈顶元素的引用，类型为 T&。如果栈为空，返回值未定义。

- **push(const T& obj)**：可以将对象副本压入栈顶。这是通过调用底层容器的 push_back() 函数完成的。

- **push(T&& obj)**：以移动对象的方式将对象压入栈顶。这是**通过调用底层容器的有右值引用参数的 push_back() 函数完成的**。

- **pop()**：弹出栈顶元素。

- **size()**：返回栈中元素的个数。

- **empty()**：在栈中没有元素的情况下返回 true。

- **emplace()**：用**传入的参数调用构造函数**，在栈顶生成对象。这意味着传入T对应的构造函数所需参数即可在栈顶生成对象，而不需要先生成一个T对象再push进去，更节省内存。

- **swap(stack<T> & other_stack)**：将当前栈中的元素和参数中的元素交换。参数所包含元素的类型必须和当前栈的相同。对于 stack 对象有一个特例化的全局函数 swap() 可以使用。注意这**实质上交换了的是指向的内存位置**，因此即使size不同也可以交换。

同时stack也**支持多种运算符**。stack<T> 模板也定义了复制和移动版的 **operator=()** 函数，因此可以将一个 stack 对象赋值给另一个 stack 对象。stack 对象有**一整套比较运算符**。比较运算通过字典的方式来比较底层容器中相应的元素。字典比较是一种用来对字典中的单词进行排序的方式。