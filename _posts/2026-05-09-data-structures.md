---
layout: post
title: 数据结构学习笔记
date: 2026-05-09 14:39:58 +0800
description: 系统整理链表、栈、队列、树、排序与图等常见数据结构的概念、复杂度和 C 语言实现。
tags: [数据结构, C语言, 算法]
categories: [学习笔记]
published: true
toc:
  sidebar: left
---

本笔记基于

[【强烈推荐】深入浅出数据结构 - 顶尖程序员图文讲解 - UP主翻译校对 (已完结)](https://www.bilibili.com/video/BV1Fv4y1f7T1/?p=4&share_source=copy_web&vd_source=539a6f293ed8a1deaa368549e9b9d579)

## 导论

ADT（abstract data type） 抽象数据结构

## 链表

主要是解决数组**内存利用率不高**和**动态性差**的问题。

链表实际上在用**空间复杂度换时间复杂度**；

![链表节点结构示意图]({{ '/assets/img/blog/data-structures/image-20260422160529077.png' | relative_url }})

定义节点的结构体如下：

```c
struct node{
 int data;
 Node *next;
}
```

链表只能从一个节点访问到下一个节点，不支持随机存储，如果想访问最后一个元素只能从头节点开始，时间复杂度高。

针对查询，时间与n呈正相关；对于插入和删除，时间复杂度最坏的情况下为O(n).

| 操作           | 数组                                                             | 链表                                     |
| -------------- | ---------------------------------------------------------------- | ---------------------------------------- |
| 访问元素的成本 | O(1)                                                             | O(n)                                     |
| 内存需求       | 大小固定，可能会有未使用的内存                                   | 用一个申请一个，但每个节点占用比数组的多 |
| 插入数据       | 1. 在开头插入 O(n)；2. 末尾 未满O(1) 已满 O(n)；3. 中间位置 O(n) | 1. O(1)；2. O(n)；3. O(n)                |
| 删除数据       | 同插入                                                           | 同插入                                   |

### C语言实现

#### 单链表

- 从头插入

```C
void InsertAtStart(int x) {
    struct Node * temp = (struct Node*)malloc(sizeof(struct Node));
    temp->data =x;
    temp->next = head;
    head = temp;
}
```

- 末尾插入

```c
void InsertAtEnd(int data){
    struct Node * temp = (struct Node*)malloc(sizeof(struct Node));
    temp->data = data;
    temp->next = NULL;
    if(head == NULL){
        head = temp;
        return;
    }
    struct Node * temp1 = head;
    while(temp1->next!=NULL){
        temp1 = temp1->next;
    }
    temp1->next = temp;
}
```

- 任意位置插入

```C
void InsertAtMiddle(int data,int n) {
    if (n < 1) {
        printf("Invalid position\n");
        return;
    }
    struct Node * temp1 = (struct Node *)malloc(sizeof(struct Node));
    if (temp1 == NULL) {
        return;
    }
    temp1->data = data;
    temp1->next = NULL;
    if (n == 1) {
        temp1->next = head;
        head = temp1;
        return;
    }
    struct Node * temp2 = head;
    for (int i = 1; i < n - 1 && temp2 != NULL; i++) {
        temp2 = temp2->next;
    }
    if (temp2 == NULL) {
        printf("Invalid position\n");
        free(temp1);
        return;
    }
    temp1->next = temp2->next;
    temp2->next = temp1;
}
```

- 删除

```C
void link_Delete(int n){
    struct Node *temp1 = (struct Node *)malloc(sizeof(struct Node));
    if (temp1 == NULL) {
        return;
    }
    temp1 = head;
    if(n==1){
        head = temp1->next;
        free(temp1);
        return;
    }
    for(int i = 1;i<n-1 && temp1!=NULL;i++){
        temp1 = temp1->next;
    }
    struct Node *temp2 = temp1->next;
    temp1->next = temp2->next;
    free(temp2);
}
```

- 反转一个链表

```C
//迭代方法
void Reverse(){
    struct Node *current ,*prev , *next;
    current = head;
    prev = NULL;
    next = NULL;
    while (current != NULL)
    {
        next = current->next;
        current->next = prev;
        prev = current;
        current = next;
    }
    head = prev;
}
//反转打印
void Print(struct Node *head){
    if(head == NULL){
        printf("\n");
        return;
    }
    printf("%d ",head->data);
    Print(head->next);
}
void PrintReverse(struct Node *head){
    if(head == NULL){
        return;
    }
    PrintReverse(head->next);
    printf("%d ",head->data);
}
//递归方法
void Reverse_Recursion(struct Node * p){
    if(p->next == NULL){
        head = p;
        return;
    }
    Reverse_Recursion(p->next);
    struct Node *q = p->next;
    q->next = p;
    p->next = NULL;
}
```

#### 双链表

```C
//头插入 尾插入 删除 打印 反转打印
#include <stdio.h>
#include <stdlib.h>

struct Node
{
    int data;
    struct Node* next;
    struct Node* prev;
};
struct Node *head;
struct Node *CreatNewNode(int x){
    struct Node * NewNode  = (struct Node *)malloc(sizeof(struct Node));
    NewNode->data = x;
    NewNode->prev = NULL;
    NewNode->next = NULL;
    return NewNode;
}

void InsertAtHead(int x){
    struct Node *temp = CreatNewNode(x);
    if (head == NULL)
    {
        head = temp;
        return;
    }
    temp->next = head;
    head->prev = temp;
    head = temp;
}

void InsertAtTail(int x){
    struct Node *temp = CreatNewNode(x);
    if (head == NULL)
    {
        head = temp;
        return;
    }
    struct Node *temp1 = head;
    while(temp1->next != NULL){
        temp1 = temp1->next;
    }
    temp1->next = temp;
    temp->prev = temp1;
}

void Delete(int n){
    struct Node *temp = head;
    if (head == NULL)
    {
        return;
    }
    for(int i=0; i<n-1; i++){
        temp = temp->next;
    }
    temp->prev->next = temp->next;
    temp->next->prev = temp->prev;
}

void Print(){
    struct Node *temp = head;
    if(temp == NULL)
    {
        return;
    }
    printf("Forward: ");
    while(temp != NULL){
        printf("%d ",temp->data);
        temp = temp->next;
    }
    printf("\n");
}
void PrintReverse(){
    struct Node *temp = head;
    if(temp == NULL)
    {
        return;
    }
    while(temp->next != NULL){
        temp = temp->next;
    }
    printf("Reverse: ");
    while(temp != NULL){
        printf("%d ", temp->data);
        temp = temp->prev;
    }
    printf("\n");
}

int main(void){
    InsertAtHead(2); Print();PrintReverse();
    InsertAtHead(4); Print();PrintReverse();
    InsertAtHead(6); Print();PrintReverse();
    InsertAtTail(8); Print();PrintReverse();
    Delete(3);       Print();PrintReverse();
    return 0;
}
```

## 栈

先进后出，后进先出（Last In First Out）

应用:

- 函数回调/递归
- undo in an editor
- 括号验证

常用操作: `push` `pop` `top` `Isempty` 每个操作的时间复杂度应该是O(1)

想要实现栈，可以用数组和链表。

使用数组时，要提前给数组分配大小，就会遇到栈溢出的问题，而使用链表则不会。

### 栈的C语言实现

```c

#include <stdio.h>
#include <stdlib.h>

struct stack{
    int data;
    struct stack *link;
};

struct stack *top = NULL;

void push(int data){
    struct stack *temp = (struct stack *)malloc(sizeof(struct stack));
    temp->data = data;
    temp->link = top;
    top = temp;
}

void pop(){
    if(top == NULL) return;
    struct stack *temp = top;
    top = top->link;
    free(temp);
}

int TOP(){
    return top->data;
}

void IsEmpty(){
    if(top == NULL){
        printf("The stack is empty!\n");
    }else{
        printf("The stack is not empty!\n");
    }
}

void Print(){
    struct stack *temp = top;
    while(temp != NULL){
        printf("%d ",temp->data);
        temp = temp->link;
    }
    printf("\n");
}
int main(){
    IsEmpty();
    push(1);Print();
    push(2);Print();
    push(3);Print();
    pop();  Print(); printf("Top element is %d\n",TOP());
    push(4);Print();
    IsEmpty();
    return 0;
}
```

### 括号匹配

123

### 前缀中缀后缀

123

## 队列

队列与栈不同，队列是FIFO (First In First OUT) 的。
其常用操作有`Enqueue`,`Dequeue`,`Isempty`,`front`等。
需要注意的是，每个操作的时间复杂度都是O(1)。

### 使用数组实现队列

在此使用环形队列，默认空队列的front和rear为-1.使用++/--时应注意为：
`front = (front + 1 ) % N;   //front++`
`front = (front - 1 + N ) % N;  //front--`
显然，此时所有的操作时间复杂度都是O(1). 如果队列已满，需要扩展时，时间复杂度提高.

```C
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

#define MAX_SIZE 100

typedef struct {
    int data[MAX_SIZE];
    int front;
    int rear;
} Queue;

bool Isempty(Queue *q){
    if(q->front == -1 && q->rear == -1){
        return true;
    }else return false;
}

bool IsFull(Queue *q){
    if((q->rear + 1) % MAX_SIZE == q->front){
        return true;
    }else return false;
}

Queue * Enqueue(Queue *q, int value){
    if(Isempty(q)){
        q->front = 0;
        q->rear = 0;
    }

    else if(IsFull(q)){
        printf("The Queue is Full already! \n");
        return q;
    }
    else{
        q->rear = (q->rear + 1) % MAX_SIZE;
    }
    q->data[q->rear] = value;
    return q;
}

Queue * Dequeue(Queue *q){
    if(Isempty(q)){
        printf("error! The Queue is Empty already! \n");
        return q;
    }
    else if(q->front == q->rear){ // 只有一个元素，出队后变空
        q->front = -1;
        q->rear = -1;
    }
    else {
        q->front = (q->front+1) % MAX_SIZE;
    }
    return q;
}

void Print(Queue *q){
    int front = q->front;
    int rear = q->rear;
    if(Isempty(q)) {
        printf("Queue is empty\n");
        return;
    }
    while((front % MAX_SIZE) != rear){
        printf("%d ", q->data[front]);
        front = (front+1) % MAX_SIZE;
    }
    printf("%d ", q->data[rear]);
    printf("\n ");
}
int main(){
    Queue *q = (Queue *)malloc(sizeof(Queue));
    q->front = -1;
    q->rear = -1;
    q = Enqueue(q, 1);
    q = Enqueue(q, 2);
    q = Enqueue(q, 3);
    q = Enqueue(q, 5);
    q = Enqueue(q, 6); // 1 2 3 5 6
    Print(q);
    q = Dequeue(q); // 2 3 5 6
    Print(q);
    q = Enqueue(q, 7); // 2 3 5 6 7
    Print(q);
    q = Dequeue(q); // 3 5 6 7
    Print(q);
    return 0;

}
```

### 使用链表实现队列

可以看出队列的实现实际上就是对一个链表进行头插和尾去(两者可以互换)，可以多使用一个`rear`标志来指引

```C
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

struct Node {
    int data;
    struct Node *next;
};

typedef struct {
    struct Node *front;
    struct Node *rear;
} Queue;

bool Isempty(Queue *q){
    if(q->front == NULL && q->rear == NULL){
        return true;
    }else return false;
}

Queue * Enqueue(Queue *q, int value){
    struct Node *newNode = (struct Node *)malloc(sizeof(struct Node));
    newNode->data = value;
    newNode->next = NULL;

    if(Isempty(q)){
        q->front = newNode;
        q->rear = newNode;
    }
    else {
        q->rear->next = newNode;
        q->rear = newNode;
    }
    return q;
}

Queue * Dequeue(Queue *q){
    struct Node *temp = q->front;
    if(Isempty(q)){
        printf("error! The Queue is Empty already! \n");
        return q;
    }
    else if(q->front == q->rear){ // 只有一个元素，出队后变空
        q->front = NULL;
        q->rear = NULL;
    }
    else {
        q->front = q->front->next;
    }
    free(temp);
    return q;
}

void Print(Queue *q){
    struct Node *current = q->front;
    if(Isempty(q)) {
        printf("The Queue is Empty already! \n");
        return;
    }
    while(current != NULL){
        printf("%d ", current->data);
        current = current->next;
    }
    printf("\n");
}

int main() {
    printf("This is a Queue implemented with Linked List! \n");
    Queue *q = (Queue *)malloc(sizeof(Queue));
    q->front = NULL;
    q->rear = NULL;
    q = Enqueue(q, 10);
    q = Enqueue(q, 20);
    q = Enqueue(q, 30);
    printf("The Queue after Enqueue: ");
    Print(q);
    q = Dequeue(q);
    q = Enqueue(q, 40);
    printf("The Queue after Dequeue: ");
    Print(q);

    return 0;
}
```

## 树

一种表示层级的数据结构
关键词：`root 根` `children 子节点` `Parent 父节点` `sibling 兄弟节点` `leaf 叶子节点` `内部节点`

- 遍历一棵树的时候是单向的
- 每个子节点向下又能视为一颗树，可以使用递归的思想
- 每有N个节点，就有N-1条边`edge`
- `Depth`节点深度为从根节点开始到该节点的边数
- `Height`节点高度是从这个节点开始到叶子节点的最远距离
- 树的高度被定义为根节点的高度

![树的节点与层级结构示意图]({{ '/assets/img/blog/data-structures/image.png' | relative_url }})

```C
struct tree{
    int data;
    struct tree *left;
    struct tree *right;
};
```

### 二叉树

设定root为第0层，从这以后的数学推论从0开始。
唯一的条件是一个节点不能有超过两个子节点

- 严格二叉树：树中的每个非叶子节点都拥有两个子节点。如果一个节点有子节点，那么它必须有两个（一左一右）。
- 完全二叉树：完全二叉树要求除最后一层外所有层填满，且最后一层节点靠左连续对齐。
- 完美二叉树：所有节点都被填满。

> - 完美二叉树最大节点数：$$2^{h+1} - 1$$
> - 完美二叉树根据节点计算高度: $$h = \log_2(n+1) - 1$$
> - 完全二叉树的高度：$$h = \lfloor \log_2n \rfloor$$
>   ![完全二叉树示意图]({{ '/assets/img/blog/data-structures/image-1.png' | relative_url }})
>   ![完美二叉树示意图]({{ '/assets/img/blog/data-structures/image-2.png' | relative_url }})
>   **在节点数一定的情况下完全二叉树的高度更小，时间复杂度更小**

平衡二叉树：计算该节点的左子树和右子树高度的差（令空子树的高度为-1），差值为不大于k。一般k = 1.

![完全二叉树高度示意图]({{ '/assets/img/blog/data-structures/image-3.png' | relative_url }}) 只针对完全二叉树成立

#### 二叉搜索树(Binary Search Tree)

|           | Array(unsorted) | Linked List | Array(sorted) | BST(balanced) |
| --------- | --------------- | ----------- | ------------- | ------------- |
| Search(x) | O(n)            | O(n)        | O(logn)       | O(logn)       |
| Insert(x) | O(1)            | O(1)        | O(n)          | O(logn)       |
| Remove(x) | O(n)            | O(n)        | O(n)          | O(logn)       |

BST定义：所有左子树上的节点值都比该节点的值要小，所有右子树上的节点值都比该节点的值要大。

#### 二叉树的遍历

![二叉树遍历顺序示意图]({{ '/assets/img/blog/data-structures/image-4.png' | relative_url }})

- 广度遍历
  - 层级遍历: F D J B E G K A C I H
    - 时间复杂度O(n) 空间复杂度最好O(1) 最差O(n) 平均O(n)

```C
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

struct TreeNode
{
    char data;
    struct TreeNode *left;
    struct TreeNode *right;
};

// 先实现一个队列
struct Node
{
    struct TreeNode *data;
    struct Node *next;
};

typedef struct
{
    struct Node *front;
    struct Node *rear;
} Queue;

bool IsEmpty(Queue *q)
{
    if (q->front == NULL && q->rear == NULL)
    {
        return true;
    }
    else
        return false;
}

Queue *Enqueue(Queue *q, struct TreeNode *data)
{
    struct Node *temp = (struct Node *)malloc(sizeof(struct Node));
    temp->data = data;
    temp->next = NULL;
    if (IsEmpty(q))
    {
        q->front = temp;
        q->rear = temp;
        return q;
    }
    else
    {
        q->rear->next = temp;
        q->rear = temp;
    }
    return q;
}

Queue *Dequeue(Queue *q)
{
    struct Node *temp = q->front;
    if (IsEmpty(q))
    {
        printf("The Queue is empty already! \n");
        return q;
    }
    else if (q->front == q->rear)
    {
        q->front = NULL;
        q->rear = NULL;
    }
    else
    {
        q->front = q->front->next;
    }
    free(temp);
    return q;
}

// 实现树的层序遍历
void LevelOrder(struct TreeNode *root)
{
    if (root == NULL)
    {
        printf("Empty tree! \n");
        return;
    }
    Queue *q = (Queue *)malloc(sizeof(Queue));
    q->front = NULL;
    q->rear = NULL;
    q = Enqueue(q, root);
    while (!IsEmpty(q))
    {
        struct TreeNode *current = q->front->data;
        if (current->left != NULL)
        {
            q = Enqueue(q, current->left);
        }
        if (current->right != NULL)
        {
            q = Enqueue(q, current->right);
        }
        printf("%c", current->data);
        q = Dequeue(q);
    }
    printf("\n");

    free(q);
}

// 辅助创建树节点的函数
struct TreeNode *createNode(char data)
{
    struct TreeNode *newNode = (struct TreeNode *)malloc(sizeof(struct TreeNode));
    newNode->data = data;
    newNode->left = NULL;
    newNode->right = NULL;
    return newNode;
}

int main()
{
    /* 构造一棵简单的树用于测试:
            A
           / \
          B   C
         / \   \
        D   E   F
    */
    struct TreeNode *root = createNode('A');
    root->left = createNode('B');
    root->right = createNode('C');
    root->left->left = createNode('D');
    root->left->right = createNode('E');
    root->right->right = createNode('F');

    printf("Level Order Traversal: ");
    LevelOrder(root); // 期望输出: A B C D E F
    return 0;
}
```

- 深度遍历
  - `<root> <left> <right>` -- 前序遍历 `DLR` FDBACEJGIHK
    - `<left> <root> <right>` -- 中序遍历 `LDR` ABCDEFGHIJK
    - `<left> <right> <root>` -- 后序遍历 `LRD` ACBEDHIGKJF
    - 时间复杂度O(n) 空间复杂度O(h) 最差O(n) 最好/平均O(logn)

```C
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

struct TreeNode
{
    char data;
    struct TreeNode *left;
    struct TreeNode *right;
};

// 辅助创建树节点的函数
struct TreeNode *createNode(char data)
{
    struct TreeNode *newNode = (struct TreeNode *)malloc(sizeof(struct TreeNode));
    newNode->data = data;
    newNode->left = NULL;
    newNode->right = NULL;
    return newNode;
}

void PreOrder(struct TreeNode *root)
{
    if (root == NULL)
        return;
    printf("%c ", root->data);
    PreOrder(root->left);
    PreOrder(root->right);
}

void InOrder(struct TreeNode *root)
{
    if (root == NULL)
        return;
    InOrder(root->left);
    printf("%c ", root->data);
    InOrder(root->right);
}

void PostOrder(struct TreeNode *root)
{
    if (root == NULL)
        return;
    PostOrder(root->left);
    PostOrder(root->right);
    printf("%c ", root->data);
}
int main()
{
    /* 构造一棵简单的树用于测试:
              F
           /    \
          D      J
         / \    / \
        B   E  G   K
       / \      \
      A   C      I
                 /
                H
    */
    struct TreeNode *root = createNode('F');
    root->left = createNode('D');
    root->right = createNode('J');
    root->left->left = createNode('B');
    root->left->right = createNode('E');
    root->left->left->left = createNode('A');
    root->left->left->right = createNode('C');
    root->right->left = createNode('G');
    root->right->left->right = createNode('I');
    root->right->left->right->left = createNode('H');
    root->right->right = createNode('K');
    printf("PreOrder: ");
    PreOrder(root);
    printf("\nInOrder: ");
    InOrder(root);
    printf("\nPostOrder: ");
    PostOrder(root);

    return 0;
}
```

#### 判断是否为二叉搜索树

1. 使用递归方法，判断左子树中的每个元素都要小于当前root节点的data，右子树每个元素都要大于当前root的data，循环执行直到跳出。

```C
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

struct Node
{
    int data;
    struct Node *left;
    struct Node *right;
};

// 辅助创建树节点的函数
struct Node *createNode(int data)
{
    struct Node *newNode = (struct Node *)malloc(sizeof(struct Node));
    newNode->data = data;
    newNode->left = NULL;
    newNode->right = NULL;
    return newNode;
}

bool IsBST(struct Node *root, int Min_value, int Max_value)
{
    if (root == NULL)
        return true;
    if (root->data > Min_value && root->data <= Max_value && IsBST(root->left, Min_value, root->data) && IsBST(root->right, root->data, Max_value))
        return true;
    else
        return false;
}

int main()
{
    /* 构造一棵简单的树用于测试:
              7
           /    \
          4      9
         / \    / \
        1   6  8   10
    */
    struct Node *root = createNode(7);
    root->left = createNode(4);
    root->right = createNode(9);
    root->left->left = createNode(5);
    root->left->right = createNode(6);
    root->right->left = createNode(8);
    root->right->right = createNode(10);
    if (IsBST(root, -2147483648, 2147483647))
        printf("The tree is a binary search tree.\n");
    else
        printf("The tree is not a binary search tree.\n");
    return 0;
}
```

1. 使用中序遍历，判断遍历后的数据是否是一个递增的数组

#### BST删除一个节点

既要删除节点，又要保证删除后的二叉树仍为BST，可以分为三种情况：

1. 删除的节点是叶子节点，直接删除，返回NULL
2. 删除的节点只有左子树或右子树，返回root.right，free当前root；
3. 删除的节点既有左子树又有右子树，寻找右子树里的最小值，替换掉root.data，然后删除右子树里的最小值。

```C
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

struct TreeNode
{
    int data;
    struct TreeNode *left;
    struct TreeNode *right;
};
// 辅助创建树节点的函数
struct TreeNode *createNode(char data)
{
    struct TreeNode *newNode = (struct TreeNode *)malloc(sizeof(struct TreeNode));
    newNode->data = data;
    newNode->left = NULL;
    newNode->right = NULL;
    return newNode;
}

struct TreeNode *FindMin(struct TreeNode *root)
{
    while (root->left != NULL)
        root = root->left;
    return root;
}

struct TreeNode *FindMax(struct TreeNode *root)
{
    while (root->right != NULL)
        root = root->right;
    return root;
}

struct TreeNode *Delete(struct TreeNode *root, int data)
{
    if (root == NULL)
        return root;
    else if (root->data < data)
    {
        root->right = Delete(root->right, data);
    }
    else if (root->data > data)
    {
        root->left = Delete(root->left, data);
    }
    else
    {
        // case 1:叶子节点
        if (root->left == NULL && root->right == NULL)
        {
            free(root);
            root = NULL;
        }
        // case 2: 只有左子树或只有右子树
        else if (root->left == NULL)
        { // 右子树
            struct TreeNode *temp = root;
            root = root->right;
            free(temp);
        }
        else if (root->right == NULL)
        { // 左子树
            struct TreeNode *temp = root;
            root = root->left;
            free(temp);
        }
        else // case 3：既有左子树又有右子树
        {
            struct TreeNode *temp;
            temp = FindMin(root->right);
            root->data = temp->data;
            root->right = Delete(root->right, temp->data);
        }
    }
    return root;
}

// 中序遍历输出（左-根-右）
void inorderPrint(struct TreeNode *root)
{
    if (root == NULL)
        return;
    inorderPrint(root->left);
    printf("%d ", root->data);
    inorderPrint(root->right);
}

int main()
{
    /* 构造一棵简单的树用于测试:
              7
           /    \
          4      9
         / \    / \
        1   6  8   10
    */
    struct TreeNode *root = createNode(7);
    root->left = createNode(4);
    root->right = createNode(9);
    root->left->left = createNode(1);
    root->left->right = createNode(6);
    root->right->left = createNode(8);
    root->right->right = createNode(10);

    // 在这里调用删除函数并测试结果
    printf("Original tree (in-order): ");
    inorderPrint(root);
    printf("\nDeleting 4...\n");
    root = Delete(root, 4);
    printf("Tree after deletion (in-order): ");
    inorderPrint(root);
    return 0;
}
```

#### BST的中序后继节点

在二叉搜索树（BST）中找特定节点的后继（Successor），最简单粗暴的方法是完整遍历一次存入数组，然后再找下一个元素，但这需要 $O(n)$ 的时间复杂度。
真正优雅且高效的做法是利用 BST 的结构特性（左 < 根 < 右），在 $O(h)$（$h$ 为树的高度）的时间内直接定位后继节点。
中序遍历的顺序是 左 -> 根 -> 右。在 BST 中，中序遍历的结果是一个严格递增的序列。因此，找 p 的中序后继，本质上就是找树中比 p->data 大的最小节点。

1. 如果 p 有右子树，它的中序后继一定在右子树里，且是右子树中data最小的节点（即右子树的最左下角的叶子节点）。
2. 如果 p 没有右子树，说明以 p 为根的局部子树已经遍历完了。我们需要向上回溯，找它的祖先。它的后继是距离它最近的、且将 p 包含在其左子树中的那个祖先节点。利用 BST 特性，我们直接从根节点往下找，只要向左拐，就记录一下当前的祖先。

```C
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

struct TreeNode
{
    int data;
    struct TreeNode *left;
    struct TreeNode *right;
};

struct TreeNode *createNode(int data)
{
    struct TreeNode *newNode = (struct TreeNode *)malloc(sizeof(struct TreeNode));
    newNode->data = data;
    newNode->left = NULL;
    newNode->right = NULL;
    return newNode;
}

struct TreeNode *Find(struct TreeNode *root, int data)
{
    while (root != NULL && root->data != data)
    {
        if (data < root->data)
        {
            root = root->left; // 目标小，去左边找
        }
        else
        {
            root = root->right; // 目标大，去右边找
        }
    }
    return root;
}

struct TreeNode *FindMin(struct TreeNode *root)
{
    if (root == NULL)
        return NULL;
    while (root->left != NULL)
        root = root->left;
    return root;
}

struct TreeNode *GetInOrderSuccessor(struct TreeNode *root, int data)
{
    if (root == NULL)
        return NULL;
    struct TreeNode *current = Find(root, data);
    if (current == NULL)
        return NULL;
    // case 1: 有右子树
    if (current->right != NULL)
    {
        return FindMin(current->right);
    }
    // case 2: 没有右子树
    else
    {
        struct TreeNode *successor = NULL;
        struct TreeNode *ancestor = root;
        while (ancestor != current)
        {
            if (data < ancestor->data)
            {
                successor = ancestor;
                ancestor = ancestor->left;
            }
            else
            {
                ancestor = ancestor->right;
            }
        }
        return successor;
    }
}

int main()
{
    /* 构造一棵简单的树用于测试:
                  7
               /    \
              4      9
             / \    / \
            1   6  8   10
        */
    struct TreeNode *root = createNode(7);
    struct TreeNode *temp;
    root->left = createNode(4);
    root->right = createNode(9);
    root->left->left = createNode(1);
    root->left->right = createNode(6);
    root->right->left = createNode(8);
    root->right->right = createNode(10);

    temp = GetInOrderSuccessor(root, 7);
    printf("%d \n", temp->data);

    return 0;
}
```

#### BST的前序后继节点

前序遍历的顺序是 根 -> 左 -> 右。找p的后继节点，进行分类讨论：

1. 如果p有左孩子，那么下一个必定访问p的左孩子。
2. 如果p没有左孩但是有右孩，那么下一个只能访问右孩。
3. 如果p是叶子节点，那么需要向上回溯，寻找没访问过的右分支，后续节点是距离 p 最近的、将 p 包含在其左子树中，并且拥有右孩子的那个祖先的右孩子。同样利用 BST 特性，从根往下找，只要向左拐且该节点有右孩子，就更新潜在的后继节点。

```C
struct TreeNode *GetPreOrderSuccessor(struct TreeNode *root, int data)
{
    if (root == NULL)
        return NULL;
    struct TreeNode *current = Find(root, data);
    if (current == NULL)
        return NULL;
    // case 1: 有左子树
    if (current->left != NULL)
    {
        return current->left;
    }
    else if (current->right != NULL)
    {
        return current->right;
    }
    else
    {
        struct TreeNode *successor = NULL;
        struct TreeNode *ancestor = root;
        while (ancestor != current)
        {
            if (data < ancestor->data && ancestor->right != NULL)
            {
                successor = ancestor;
                ancestor = ancestor->left;
            }
            else
            {
                ancestor = ancestor->right;
            }
        }
        return successor->right;
    }
}
```

## 图

图是由一个顶点集V和一个边集E组成的有序对组成的. 表示为$G = (V,E)$
![图的顶点与边示意图]({{ '/assets/img/blog/data-structures/image-5.png' | relative_url }})
![有向图与无向图示意图]({{ '/assets/img/blog/data-structures/image-6.png' | relative_url }})

- 根据图里的边有向还是无向可以分成**有向图(Digraph)**和**无向图**.
- 加权图:每条边赋予不同的权重,非加权图可以看成边权重全为1的加权图.
- 自环(self-loop):一条边只包含一个顶点，例如网页刷新。
- 多重边(multiedge): 两个顶点之间不止一条边，例如两地之间的航班。
- 在**没有自环和多重边**的情况下，如果$|V| = n $ ,则有向图的边数 $0 \leq |E| \leq n(n-1) $ ;无向图的边数为 $ 0 \leq |E| \leq \frac{n(n-1)}{2} $
  - 稠密的图：边的数量接近顶点数$ n^2 $ ，使用**邻接矩阵**处理
  - 稀疏的图：边的数量接近顶点数$ n $ ，使用**邻接表**处理
  - 稀疏还是稠密没有明确的界限，取决于上下文

- 简单路径（simple **Path**）：无重复的边或者顶点
- 连接：在无向图里，能从任意一个顶点走到另一个任意节点，称为连接；在有向图里，则被称为强连接。
- 闭合途径：指开始和结束在同一个顶点，且路径长度**大于0**；

### 使用C实现图

空间复杂度 $ O(|V| + |E|) $

#### 邻接矩阵

![图的邻接矩阵表示]({{ '/assets/img/blog/data-structures/image-7.png' | relative_url }})
为了表示权重图可以将1换成权重值，0换成∞。
找到相邻节点的时间花费为$ O(V) $；在给出索引的情况下，判断两个相邻节点是否相连的花费为$ O(1) $。
邻接矩阵会占用很大的内存空间，对稠密的图的表现是好的。

#### 邻接表

![图的邻接表示示意图]({{ '/assets/img/blog/data-structures/image-8.png' | relative_url }})
![图的邻接表表示]({{ '/assets/img/blog/data-structures/image-9.png' | relative_url }})
