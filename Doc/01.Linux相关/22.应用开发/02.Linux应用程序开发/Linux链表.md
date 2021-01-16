# Linux链表

## container_of(ptr,type,member)函数简介

`container_of`位于内核源码`<linux/list.h>`中，函数是内核空间的函数，所以我们在应用程序中，也就是用户态是无法使用。解决方法就是如果想使用，就在自己的应用程序下新建该函数即可，同理内核链表`list`也是无法在用户程序使用，如果应用程序想使用，就拷一份到自己的工程目录下。

问题：如何通过一个结构体成员变量，得到该结构体变量的首地址？

```c
#include <stdio.h>

struct MY_STRUCT
{
	int i;
	int j;
	int k;
};

struct MY_STRUCT test={
    1,2,3
};
/*
 * 假如我们知道test.k的地址，如果求得，test的地址呢？
 */


int main(int argc, char *argv[])
{
	MY_STRUCT *p = container_of(&test.k,struct MY_STRUCT,k);
    printf("p addr:%d test addr:%d \n",(uint32_t)p,(uint32_t)test);
    
}


```

