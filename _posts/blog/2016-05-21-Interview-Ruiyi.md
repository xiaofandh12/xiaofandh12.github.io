---
layout: post
title: 面试题 -- 成都睿艺科技有限公司
description: 面试题，睿艺，游戏
category: blog
---

## 大致流程

* 14：00 - 15：00 笔试，笔试题会在题目解析部分说明
* 15：00 - 16：10 技术面试，Bill（公司CTO）
* 16：10 - 17：30 人力面试，老杨（HR）

## 题目解析

### 编程题（4选3）
1. 已知带头节点的循环双链表节点定义为：
	
	```
	struct list_node {
	struct list_node * next, *prev;
	};
	```
	实现双链表插入和删除函数：
	//新节点new插入在节点node后面
	
	```
	void  list_insert(struct list_node *node, struct list_node *new);
	```
	
	//从链表中删除node节点
	
	```
	void list_delete(struct list_node * node);
	```

	解答：

	```
	void list_insert(struct list_node *node, struct list_node *new) {
		if (new == NULL) return;
		if (node == NULL) node = new;
		else {
			list_node * temp = node->next;
			node->next = new;
			new->next = temp;
			new->prev = node;
			temp->prev = new;
		}
	}
	
	void list_delete(struct list_node *node) {
		if (node == NULL) return;
		if (node->newxt == node) node = NULL;
		else if {
			list_node * temp1 = node->prev;
			list_node * temp2 = node->next;
			temp1->next = temp2;
			temp2->prev = temp1;
		}
	}
	```
	
2. 已知二叉树节点定义为：

	```
	typedef struct bittree_node {
		int val;
		BiTNode * lchild;
		BiTNode * rchild;
	}BiTNode;
	```
	实现二叉树的广度优先遍历，遍历过程中打出每个节点的值。二叉树假设以及造好，若用到辅助数据结构（如栈，队列）可用伪码。
	
	```
	void BFS(BiTNode *pRoot);
	```
	
	解答：
	
3. 环形缓冲区是生产者和消费者模型中常用的数据结构。生产者将数据放入数组的尾端，而消费者从数组的另一端移走数据，当达到数组的尾部时，生产者绕回到数组的头部。如图所示：

	![image](/images/2016-05-21-Interview-Ruiyi/CircularBuff.jpg)

	定义循环缓冲队列如下（省略其他函数，并假设数据已初始化好），请实现WriteData()和ReadData()方法(二选一):

	```
	class CircularBuff
	{
	public:
	//...
	
	//返回实际写入的字节数
	int WriteData(const char *pBuff, int iLen);
	
	//返回实际读到的字节数
	int ReadData(char * pBuff, int iLen);
	
	private:
		int m_iRead;
		int m_iWrite;
		int m_iSize;//buff的大小
		char * m_pBuff;
	}
	```

	解答：

	```
	int WriteData(const char * pBuff, int iLen) {
		int res = 0;
		for (int i = 0; i < iLen; i++) {
			if ((m_iWrite + 1) % m_buff) == m_iRead) return res;
			else {
				*(m_pBuff + m_iWrite) = *(pBuff + 1);
				m_iWrite = (m_iWrite + 1) % m_buff;
				res++;
			}
		}
	}
	```
	

4. 内存中有1万个整数的数组，写一段算法，找出前100个最大的整数（可用伪码）。

	```
	/#define RES_ARR_SIZE 100
	/#define ARR_SIZE 10000
	//Arr: 输入数组， ResArr: 结果数组
	void FindMax(int Arr[], int ResArr[]);
	```
 
 	解答：
 	
 	```
 	
 	```
 	
### 问答题（6选5）:

1. 请解释继承、组合、函数重载（Overload）、函数重构（override）的概念。

	解答：继承 - 之类拥有父类的方法、成员变量，is-a；组合 - has-a；重载 - 同一个函数拥有不同的参数列表；重构 - 子类覆写父类的方法，该方法需要为virtual。

2. C++中replacement new和new的区别。

	解答：replacement new的存储地址已提前申请好，只是从地址池传一个指针过去即可，这样可以避免内存的频繁申请和释放，产生内存碎片。
	
3. 进程间通信主要包含哪几种方式，请简要说明。

	解答：管道(Pipe)，信号量(Semophore)，信号(Singal)，消息队列(Message Queue)，共享内存(Shared Memory)，套接字(Socker)

4. 什么是守护（daemon）进程？列出编写守护进程的主要步骤。
	
	解答：守护进程
	
5. 已知mysql一张数据表最多存1千万条数据，假设用户数据存于数据库GameDB中的AccountTable，在游戏运营过程中，注册用户数会不断增加，假设设置一张表800万条数据为警戒线，达到警戒线数据库表需要扩容，请简述你的扩容方案。

	解答：分裂法，先停服，把一张表格复制两份（同一个数据库，或不同数据库，或不同机器），然后规定第一张表格存储奇数ID的数据，第二张表格存储偶数ID的数据，然后使用第三方的工具删除第一张表格的偶数ID的数据，删除第二张表格奇数ID的数据。

6. 在玩家对战的实时交互中，拟使用UDP进行网络通信，但需要在应用层解决UDP不可靠（乱序、丢包）的问题，请简述你的可靠UDP方案（最好图示之）。

	* 解答： UDP报文中加入序列号，校验码；发送端对每一个序列号都要求接收端发送ACK，如果没有收到ACK，就重发该序列号的报文，接收端如果收到重复报文就丢弃。
	
## 技术面试

做完题之后，Bill主要就题目跟我做了一些讨论，也大概问了一下我以前做的事情，最后给了我一些学习的建议，如果去了它们公司会怎么样的培养，会让我在实际的项目中获得很大的成长。

我题目做的不是很好，广度优先搜索不会写，写了一个深度优先搜索，Bill基本把我不会做的题，都给我讲了一遍，最后给我的评价是：具备基本的编程能力，对概念的理解不够深入，看书要带着问题去看，对实际问题的解决方案有待加强。推荐我要好好加强基础学习：C++，设计模式（着重于理解），数据结构（《数据结构（c语言版）》严蔚敏或《数据结构（C++语言版）》）；对新东西或自己不了解的领域，要在心中有个大的图景，然后看看人家是怎么解决的，一步步越来越细的学习，在实际中去领会，最终要使之成为自己的知识。

## 人力面试

和Bill聊完之后，Bill就让我等一下，HR再来和我聊聊，过了一会儿老杨就来了。他先是了解了一些我的基本情况，然后介绍了一下公司的气质、一些制度、公司的基本情况，工资要求等。

问了一下我的基本情况：

* 哪里毕业的？
* 在人人网、三星实习的情况？
* 为什么想要离开现在的公司？
* 在现在的公司处于什么的位置、作用？
* 觉得我的专业和华为很对口，为什么没去华为？
* 为什么想去游戏行业？

公司的气质：学习和分享。

公司的制度：

* 周六加班每个月不超过两次；
* 实习期两个月，实习期领实习工资，转正后领转正工资；
* 转正后六个月会迎来重大的加薪机会，八个月的时间可以考察出一个人的天花板在哪里，以此来定；
* 以后每半年都会有是否加薪的评定；
* 公司没有负激励，只有正激励：如早上按要求是需要九点到，但如果到不了，只需要在公司的群里面说一声即可，不会有什么惩罚，但也起到了大家共同监督的作用；如果个人加班是没有什么奖励的，如果小团队加班是会记录在案，如果所有人加班也是会记录在案，年终有一项奖金是和加班相关的；等等吧。

公司的基本情况：

* 公司目前25人，7个程序，后端主要使用C++，然后使用Unity3D生成Android和IOS客户端。
* 公司CTO为Bill，HR为老杨，还有一个制片人，然后就是程序、策划、美术