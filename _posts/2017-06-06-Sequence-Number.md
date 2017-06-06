---
layout: post
title: Sequence Number
categories: ACM
---

> NEUOJ  
> 2017.4.29--2017年新生组队赛（二）   

<!-- more -->

__Link: [https://oj.neu.edu.cn/problem/1163](https://oj.neu.edu.cn/problem/1163)__  

### Problem Description
In Linear algebra, we have learned the definition of inversion number:  
Assuming A is a ordered set with n numbers ( n > 1 ) which are different from each other. If exist positive integers i , j, ( 1 ≤ i ＜ j ≤ n and A[i] ＞ A[j]), <A[i], A[j]> is regarded as one of A’s inversions. The number of inversions is regarded as inversion number. Such as, inversions of array <2,3,8,6,1> are <2,1>, <3,1>, <8,1>, <8,6>,<6,1>,and the inversion number is 5.  
Similarly, we define a new notion —— sequence number, If exist positive integers i, j, ( 1 ≤ i ≤ j ≤ n and A[i]  <=  A[j], <A[i], A[j]> is regarded as one of A’s sequence pair. The number of sequence pairs is regarded as sequence number. Define j – i as the length of the sequence pair.  
Now, we wonder that the largest length S of all sequence pairs for a given array A.  

### Input
There are multiply test cases.  
In each case, the first line is a number N(1<=N<=50000 ), indicates the size of the array, the 2th ~n+1th line are one number per line, indicates the element Ai （1<=Ai<=10^9） of the array.  

### Output
Output the answer S in one line for each case.  

### Sample Input
5  
2 3 8 6 1  

### Sample Output
3  

<hr/>

### 分析
题目中给出一个数组a，让求满足 a[i] <= a[j] 的 j - i 最大值，即最大距离。  

一个数y如果大于等于它左边的数x，那么当再出现更大的数z时，z到x的距离一定大于z到y的距离，因此y就不需要保留，只需要保留y到它前面比它小的数的最大距离即可；  
一个数y如果小于它左边的数x，就保留下来。  

这里使用类似于“栈”的结构：当新读入的数小于栈顶的数时，新读入的数入栈；当新读入的数大于等于栈顶的数时，计算新读入的数与栈中比他小的数的距离，只保留最大距离即可。最后结果就是最大的距离。  

### 代码
{% highlight c++ %}
#include<cstdio>
#include<iostream>

using namespace std;

struct mystack
{
    int num;
    int position;
}a[50010];

int main(void)
{
    int n, i, j, h, temp, maxlen;
    while(scanf("%d", &n) != EOF)
    {
        h = 0;
        maxlen = 0;
        for(i = 0;i < n;i++)
        {
            scanf("%d", &temp);
            if(!i || (i && temp < a[h-1].num))
            {
                a[h].num = temp;
                a[h].position = i;
                h++;
            }
            else
            {
                for(j = 0;j < h;j++)
                {
                    if(temp >= a[j].num)
                    {
                        maxlen = max(i - a[j].position, maxlen);
                    }
                }
            }
        }
        printf("%d\n", maxlen);
    }
    return 0;
}
{% endhighlight %}
