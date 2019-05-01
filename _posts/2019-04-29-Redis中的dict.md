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
      dictType *type;   // 哈希表类型
      void *privdata;   // 哈希表的一些私有数据
      dictht ht[2];     // 两个具体的哈希表，ht[0]是主要使用的表，ht[1]在rehash的时候会用到
      long rehashidx;   // rehash进行到的位置，-1表示没有在进行rehash
      unsigned long iterators; /* number of iterators currently running */
  } dict;
  ```
* `dictType`:  
  ```c
  typedef struct dictType {
      uint64_t (*hashFunction)(const void *key);            // 哈希函数
      void *(*keyDup)(void *privdata, const void *key);     // key复制函数
      void *(*valDup)(void *privdata, const void *obj);     // value复制函数
      int (*keyCompare)(void *privdata, const void *key1, const void *key2);    // key比较函数
      void (*keyDestructor)(void *privdata, void *key);     // key销毁函数
      void (*valDestructor)(void *privdata, void *obj);     // value销毁函数
  } dictType;
  ```
* `dictht`:  
  ```c
  typedef struct dictht {
      dictEntry **table;        // dictEntry指针的一维数组
      unsigned long size;       // table一维数组的长度，即该具体哈希表大小
      unsigned long sizemask;   // 哈希表大小掩码
      unsigned long used;       // 哈希表已使用大小
  } dictht;
  ```
* `dictEntry:`  
  ```c
  typedef struct dictEntry {
      void *key;                // 哈希表中该项的key
      union {
          void *val;
          uint64_t u64;
          int64_t s64;
          double d;
      } v;                      // 哈希表中该项的key
      struct dictEntry *next;   // 哈希表中另一项的指针
  } dictEntry;
  ```

用图来表示的话，整体结构是这样的：  

![dict](/public/image/dict.svg)  
{: align="center"}

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
试着模仿了一下，感觉这种写法有点像C++中的类：  
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
实际作用，就是通过`dictType`指定不同的哈希、键值函数，使得可以使用`dict`这一个结构存储多种数据，方便了代码的复用。

`dict`里的`dictht ht[2];`，可以存放两个哈希表。正常使用时主要使用第一个，当哈希表中已用的位置过多，哈希效率下降时，会触发rehash，拓展该哈希表大小并将表中的原内容迁移到新的位置上。

`dictht`里的`dictEntry **table;`，是`dictEntry指针`的一维数组，里面的每一个`dictEntry指针`都是一个单向链表的头结点，即采用了链地址法解决遇到的哈希冲突，实际的结构如下图所示：  

![dictht](/public/image/dictht.svg)  
{: align="center"}

像`src/dict.c`中的`dictFind()`函数，作用是根据键查找值，其实就是先计算key的哈希值，跟哈希表的掩码进行与操作后得到索引idx，再到`table[idx]`开头的单向链表中去遍历查找。`dictAdd()`函数作用是向哈希表中添加项，实现是先计算key的哈希值，根据掩码得到索引，然后为新项分配一块空间，放在对应位置，再填上value，如果key已存在，该操作则会退出。

rehash是由`dictExpand()`触发的，触发条件可以看`src/dict.c`中的`_dictExpandIfNeeded()`，如果哈希表为空，会调用`dictExpand()`进行初始化；如果哈希表`dictht`的used/size比达到1:1，也会调用`dictExpand()`。used是哈希表已使用的大小，即存储的`dictEntry`个数；size只是一维数组table的长度，所以这里的1:1并不是table被占满的情况，很可能在table被占满之前就已经调用了`dictExpand()`。  

`dictExpand()`扩展到的大小的话，是比指定的size大的且最接近的2^n。源代码中`DICT_HT_INITIAL_SIZE`被设定为4，表示初始化时是4，后面的扩展是used*2，比如当前size和used都为8，调用`dictExpand(d, 8*2)`，新的哈希表size就是16，代码中使用`_dictNextPower()`计算最接近的2^n。

选择2^n，则是为了得到一个合适的掩码。普通情况下，哈希函数得到一个key的哈希值，需要用它去取模哈希表的长度，得到在哈希表中的实际位置，而redis中使用2^n作为size，2^n-1作为对应的掩码，通过与操作即可替代频繁使用的取模操作，对性能的提升是很有帮助的。  

rehash的具体过程分析。  
