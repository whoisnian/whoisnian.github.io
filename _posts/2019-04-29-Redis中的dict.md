---
layout: post
title: Redis中的dict
categories: Programming
---

> 在查看Redis源码的过程中可以发现，各种命令的实现基本都用到了`src/db.c`中对`db`操作的封装，而对`db`的各种操作则又用到了`src/dict.c`中对`dict`操作的封装。  
> Redis全称`Remote Dictionary Server`，很明显，`dict`是Redis中极为重要的数据结构，这里就对它来做一个简单的分析。  

<!-- more -->

先来看一下`src/dict.h`中的有关结构：  
* `dict`:  
  ```c
  typedef struct dict {
      dictType *type;
      void *privdata;
      dictht ht[2];
      long rehashidx; /* rehashing not in progress if rehashidx == -1 */
      unsigned long iterators; /* number of iterators currently running */
  } dict;
  ```
* `dictType`:  
  ```c
  typedef struct dictType {
      uint64_t (*hashFunction)(const void *key);
      void *(*keyDup)(void *privdata, const void *key);
      void *(*valDup)(void *privdata, const void *obj);
      int (*keyCompare)(void *privdata, const void *key1, const void *key2);
      void (*keyDestructor)(void *privdata, void *key);
      void (*valDestructor)(void *privdata, void *obj);
  } dictType;
  ```
* `dictht`:  
  ```c
  typedef struct dictht {
      dictEntry **table;
      unsigned long size;
      unsigned long sizemask;
      unsigned long used;
  } dictht;
  ```
* `dictEntry:`  
  ```c
  typedef struct dictEntry {
      void *key;
      union {
          void *val;
          uint64_t u64;
          int64_t s64;
          double d;
      } v;
      struct dictEntry *next;
  } dictEntry;
  ```

用图来表示的话，整体结构是这样的：  
![dict](/public/image/dict.svg)  

这里的`dictType`看起来有点意思，它是一个由多个函数指针组成的结构体，`src/server.c`中用它去声明了一些基本的`dictType`，如：  
```c
dictType hashDictType = {
    dictSdsHash,                /* hash function */
    NULL,                       /* key dup */
    NULL,                       /* val dup */
    dictSdsKeyCompare,          /* key compare */
    dictSdsDestructor,          /* key destructor */
    dictSdsDestructor           /* val destructor */
};
```
试着模仿了一下：  
```c
#include<stdio.h>

typedef struct Animal
{
    char name[20];
    void (*print)(struct Animal *self, char *str);
}Animal;

void catprint(struct Animal *self, char *str)
{
    printf("%s: MEW~~ %s", self->name, str);
}

int main(void)
{
    Animal tom = {
        "Tom",
        catprint
    };
    char str[20] = "I am a cat.\n";
    tom.print(&tom, str);

    return 0;
}
```

未完待续。
