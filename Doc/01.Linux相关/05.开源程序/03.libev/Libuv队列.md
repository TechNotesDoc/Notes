# Libuv队列

## 简述

`libuv`中的队列使用宏定义封装而成双向循环队列, 和Linux内核链表用法一样，Linux内核节点是个结构体，成员为 `next`, `prev` 指针，而libuv队列里面使用一个数组`typedef void *QUEUE[2]`，QUEUE[0]指向下一个，QUEUE[1]指向前一个，其实效果是一样的。

## 实现原理

参见`libuv\src\queue.h`文件。`libuv` 的 queue 实现了下面的方法：

- `QUEUE_NEXT`：得到q的前一个节点

  ```c
  #define QUEUE_NEXT(q)       (*(QUEUE **) &((*(q))[0]))
  ```

- `QUEUE_PREV`：得到q的前一个节点

  ```c
  #define QUEUE_PREV(q)       (*(QUEUE **) &((*(q))[1]))
  ```

- `QUEUE_INIT`：初始化一个节点

  源码如下：

  ```c
  /* 把 q的前一个和后一个都指向自己 */
  #define QUEUE_INIT(q)                                                         \
    do {                                                                        \
      QUEUE_NEXT(q) = (q);                                                      \
      QUEUE_PREV(q) = (q);                                                      \
    }                                                                           \
    while (0)
  ```

  应用举例：

  ```c
  struct Student{
      int age;
      QUEUE node; // 节点
  };
  struct Student stu1;
  
  QUEUE g_queue_head; 			// 定义一个全局节点,作为链表头
  QUEUE_INIT(&g_queue_head);		// 初始化全局节点
  QUEUE_INIT(&stu1.node); 		// 初始化stu1的节点
  ```

-  `QUEUE_PREV_NEXT` :得到q前一个的下一个

  ```c
  #define QUEUE_PREV_NEXT(q)  (QUEUE_NEXT(QUEUE_PREV(q)))
  ```

- `QUEUE_NEXT_PREV`:得到q下一个的前一个

  ```c
  #define QUEUE_NEXT_PREV(q)  (QUEUE_PREV(QUEUE_NEXT(q)))
  ```

  

- `QUEUE_DATA`:更具节点q的地址得到节点q所在对象的首地址

  ```c
  #define QUEUE_DATA(ptr, type, field)                                          \
    ((type *) ((char *) (ptr) - offsetof(type, field)))
  ```

  应用举例：

  ```c
  struct Student{
      int age;
      QUEUE node; // 节点
  };
  struct Student stu1;
  
   
  QUEUE_INIT(&stu1.node); 
  
  // 根据&stu1.node节点地址  找到该节点所在对象的首地址
  struct Student *pStu  = QUEUE_DATA(&stu1.node,struct Student,node); 
  pStu->age = 10;
  
  ```

  

- `QUEUE_FOREACH` ：从节点p开始，遍历队列头h中所有的子节点，

  ```c
  #define QUEUE_FOREACH(q, h)                                                   \
    for ((q) = QUEUE_NEXT(h); (q) != (h); (q) = QUEUE_NEXT(q))
  ```

  

- `QUEUE_EMPTY`：判断是否为空, 空的话返回1，否则返回0

  ```c
  #define QUEUE_EMPTY(q)                                                        \
    ((const QUEUE *) (q) == (const QUEUE *) QUEUE_NEXT(q))
  
  ```

  

- `QUEUE_HEAD`：从队列头q，得到该队列下的第一个节点

  ```c
  #define QUEUE_HEAD(q)                                                         \
    (QUEUE_NEXT(q))
  ```

  

- `QUEUE_ADD`:

  ```c
  #define QUEUE_ADD(h, n)                                                       \
    do {                                                                        \
      QUEUE_PREV_NEXT(h) = QUEUE_NEXT(n);                                       \
      QUEUE_NEXT_PREV(n) = QUEUE_PREV(h);                                       \
      QUEUE_PREV(h) = QUEUE_PREV(n);                                            \
      QUEUE_PREV_NEXT(h) = (h);                                                 \
    }                                                                           \
    while (0)
  ```

  

- `QUEUE_SPLIT`：

  ```c
  #define QUEUE_SPLIT(h, q, n)                                                  \
    do {                                                                        \
      QUEUE_PREV(n) = QUEUE_PREV(h);                                            \
      QUEUE_PREV_NEXT(n) = (n);                                                 \
      QUEUE_NEXT(n) = (q);                                                      \
      QUEUE_PREV(h) = QUEUE_PREV(q);                                            \
      QUEUE_PREV_NEXT(h) = (h);                                                 \
      QUEUE_PREV(q) = (n);                                                      \
    }                                                                           \
    while (0)
  ```

  

- `QUEUE_INSERT_HEAD`：在队列头插入一个节点q

  ```c
  #define QUEUE_INSERT_HEAD(h, q)                                               \
    do {                                                                        \
      QUEUE_NEXT(q) = QUEUE_NEXT(h);                                            \
      QUEUE_PREV(q) = (h);                                                      \
      QUEUE_NEXT_PREV(q) = (q);                                                 \
      QUEUE_NEXT(h) = (q);                                                      \
    }                                                                           \
    while (0)
  ```

  

- `QUEUE_INSERT_TAIL`：在队列尾插入一个节点q

  ```
  #define QUEUE_INSERT_TAIL(h, q)                                               \
    do {                                                                        \
      QUEUE_NEXT(q) = (h);                                                      \
      QUEUE_PREV(q) = QUEUE_PREV(h);                                            \
      QUEUE_PREV_NEXT(q) = (q);                                                 \
      QUEUE_PREV(h) = (q);                                                      \
    }                                                                           \
    while (0)
  ```

  

- `QUEUE_REMOVE` :移除一个节点 q

  ```c
  #define QUEUE_REMOVE(q)                                                       \
    do {                                                                        \
      QUEUE_PREV_NEXT(q) = QUEUE_NEXT(q);                                       \
      QUEUE_NEXT_PREV(q) = QUEUE_PREV(q);                                       \
    }                                                                           \
    while (0)
  ```

## 实例代码

```c
#include <iostream>
#include <cstring>
#include "queue.h"
#include <stdio.h>
using namespace std;

struct Student{
    int age;
    char *name;
    QUEUE node;
};

int main() {
    Student user_s1;
    Student user_s2;
    user_s1.age = 10;
    user_s1.name = new char[10];
    strcpy(user_s1.name, "wesley");
    user_s2.age = 15;
    user_s2.name = new char[10];
    strcpy(user_s2.name, "Bob");

    QUEUE queue;
    QUEUE_INIT(&queue);
    QUEUE_INIT(&user_s1.node);
    QUEUE_INIT(&user_s2.node);
    QUEUE_INSERT_TAIL(&queue, &user_s1.node);
    QUEUE_INSERT_TAIL(&queue, &user_s2.node);


    QUEUE *p;
    p = QUEUE_HEAD(&queue);
    /**
    * Should retrieve the user behind the "p" pointer.
    */
    Student *first_stu = QUEUE_DATA(p, struct Student, node);

    /**
    * Should output the name of wesley.
    */
    printf("Received first inserted Student: %s who is %d.\n",
           first_stu->name, first_stu->age);

    QUEUE_FOREACH(p, &queue){
        Student *tmp = QUEUE_DATA(p, struct Student, node);
        cout<<"name: "<<tmp->name<<" age: "<<tmp->age<<endl;
    }
    return 0;
}

```

输出是:

```
Received first inserted Student: wesley who is 10.
name: wesley age: 10
name: Bob age: 15
```