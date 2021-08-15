---
title: '数据结构常见代码汇总（考研向）'
date: 2020-09-21 21:10:11
tags: []
published: true
hideInList: false
feature: 
isTop: false
---
数据结构考验中常用操作的伪代码，大部分参考王道书上代码。另有部分经典算法题代码（持续更新）。

啊 其实已经写好了, 但是懒得更新在博客了，有需要的自己去overleaf下。[打印版代码](https://www.overleaf.com/read/cykwcdbmmnyj)

<!-- more -->

@[TOC]
# 线性表
## 顺序表
 **顺序表定义**
静态分配
```C
#define Maxsize 50 
typedef struct{
    ElemType data[MaxSize];
    int length;
}SqList;
```
动态分配
```C
#define InitSize 100        //表长度的初始化定义
typedef struct {            //动态分配数组顺序表的定义
    ElemType *data;            //指示动态分配数组的指针
    int MaxSize, length;    //数组的最大容量和当前个数
} SeqList;                    //动态分配数组顺序表的类型定义
```

 **顺序表基本操作**
插入
```C
bool ListInsert(SqList &L,int i,ElemType e){
//本算法实现将元素e插入到顺序表L中第i个位置
    if(i<1||i>L.length+1)        //判断i的范围是否有效
        return false;
    if(L.length>=MaxSize)        //当前存储空间已满，不能插入
        return false;
    for(int j=L.length;j>=i;j--)//将第i个元素及之后的元素后移
        L.data[j]=L.data[j-1];
    L.data[i-1]=e;                //在位置i处放入e
    L.length++;                    //线性表长度加1
    return true;
}
```
 删除操作
```C
bool ListDelete(SqList &L,int i,ElemType &e){
//本算法实现删除顺序表L中第i个位置的元素
    if(i<1||i>L.length)            //判断i的范围是否有效
        return false;
    e=L.data[i-1];                //将被删除的元素赋值给e
    for(int j=i;j < L.length;j++)//将第i个位置之后的元素前移
        L.data[j-1]=L.data[j];
    L.length--;                    //线性表长度减1
    return true;
}
```

 按值查找
```C
int LocateElem(SqList L,ElemType e){
//本算法实现查找顺序表中值为e的元素，如果查找成功，返回元素位序，否则返回0
    int i;
    for(i=0;i < L.length;i++)
        if(L.data[i]==e)
            return i+1;            //下标为i的元素值等于e，返回其位序i+1
    return 0;                    //退出循环，说明查找失败
}
```

## 单链表
 **单链表中结点类型的描述**
```C
typedef struct LNode{            //定义单链表结点类型
    ElemType data;                //数据域
    struct LNode *next;            //指针域
}LNode,*LinkList;
```

 **单链表基本操作**
头插法
```C
LinkList CreateList1(LinkList &L){
//从表尾到表头逆向建立单链表L，每次均在头结点之后插入元素
    LNode *s;int x;
    L=(LinkList)malloc(sizeof(LNode));    //创建头结点
    L->next=NULL;                        //初始化为空链表
    while(scanf("%d",&x) != EOF){                        //循环输入
        s=(LNode*)malloc(sizeof(LNode));//创建新结点
        s->data=x;
        s->next=L->next;
        L->next=s;                        //将新结点插入表中，L为头指针
        scanf("%d",&x);
    }//while结束
    return L;
}
```
尾插法
```C
LinkList CreatList2(LinkList &L) {
//从表头到表尾正向建立单链表L，每次均在表尾插入元素
    int x;                        //设元素类型为整型
    L = (LinkList)malloc(sizeof(LNode));
    LNode *s, *r = L;            //r为表尾指针
    while(scanf("%d",&x) != EOF) {            //循环输入
        s = (LNode*)malloc(sizeof(LNode));
        s->data = x;
        r->next = s;
        r = s;                    //r指向新的表尾结点
        scanf("%d",&x);
    }
    r->next = NULL;                //尾结点指针置空
    return L;
}
```

按序号索值
```C
LNode *GetElem(LinkList L,int i){
//本算法取出单链表L(带头结点)中第i个位置的结点指针
    int j=1;            //计数，初始化为1
    LNode *p=L->next;    //头结点指针赋给P
    if(i==0)
        return L;        //若i等于0.则返回头结点
    if(i<1)
        return NULL;    //若i无效，则返回NULL
    while(p&&j<i){        //从第1个结点开始找，查找第i个结点
        p=p->next;
        j++;
    }
    return p;            //返回第i个结点的指针，如果i大于表长，p=NULL,直接返回p即可
}
```

按值查找结点
```C
LNode *LocateElem(LinkList L,ElemType e){
//本算法查找单链表L(带头结点)中数据域值等于e的结点指针，否则返回NULL
    LNode *p=L->next;
    while(p!=NULL&&p->data!=e)//从第1个结点开始查找data域为e的结点
        p=p->next;
    return p;                 //找到后返回该结点指针，否则返回NULL
}
```
插入节点操作(两种)

前插
```C
p=GetElem(L,i-1);        //查找插入位置的前驱结点
s->next=p->next;        //图2-7中操作步骤1
p->next=s;                //图2-7中操作步骤2
```
后插
```C
//将*s结点插入到*p之前的主要代码片段
s->next=p->next;        //修改指针域，不能颠倒
p->next=s;
temp=p->data;            //交换数据域部分
p->data=s->data;
s->data=temp;
```

删除节点操作
```C
//删除所给节点后一节点
p=GetElem(L,i-1);        //查找删除位置的前驱结点
q=p->next;                //令q指向被删除结点
p->next=q->next;        //将*q结点从链中“断开”
free(q);                //释放结点的存储空间
//删除所给节点
q=p->next;                //令q指向*p的后继结点
p->data=p->next->data;    //和后继结点交换数据域
p->next=q->next;        //将*q结点从链中“断开”
free(q);                //释放后继结点的存储空间
```

## 双链表
**双链表的定义**
```C
typedef struct DNode{           //定义双链表结点类型
    ElemType data;                //数据域
    struct DNode *prior,*next;    //前驱和后继指针
}DNode,*DLinkList;
```
 **双链表的基本操作**
双链表的插入操作
```C
s->next=p->next;                //将结点*s插入到结点*p之后
p->next->prior=s;
s->prior=p;
p->next=s;
```

双链表的删除操作
```C
p->next=q->next;                //图2-11中步骤1
q->next->prior=p;                //图2-11中步骤2
free(q);        
```

## 静态链表
 **静态链表的定义**
```C
#define MaxSize 50                //静态链表的最大长度
typedef struct{                    //静态链表结构类型的定义
    ElemType data;                //存储数据元素
    int next;                    //下一个元素的数组下标
}SLinkList[MaxSize];
```