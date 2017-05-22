---
layout: post
title: First_Blood
categories: ACM
---

> NEUOJ  

<!-- more -->

__Link: [https://oj.neu.edu.cn/problem/107](https://oj.neu.edu.cn/problem/107)__  

### Problem Description
NEUACMer is good at solving the "water" problems, I know. Now, there is a easy issue, called First Blood. Given two integers a, b. Return the result of a mod b.  

### Input
There are mutiple cases. Following each test cases, one line contains two integers a, b. a <= 10^100, b<=10^9.  

### Output
For each test cases, output a line for the result.  

### Sample Input
100 3  

### Sample Output
1  

<hr/>

### 分析
范围为10^100，所以只能用字符串来储存数据，大数取余，要用到同余关系的相关内容。  
例如计算`9876 % 7`：  
```
9 % 7 = 2;  
(2 * 10 + 8) % 7 = 0;  
(0 * 10 + 7) % 7 = 0;  
(0 * 10 + 6) % 7 = 6;  
```
所以最终`9876 % 7 = 6`。  

### 代码
{% highlight c++ %}
#include<cstdio>
#include<cstring>

int main(void)
{
	char a[1000];
	int b, t, i, ans;
	while(scanf("%s%d", a, &b) != EOF)
	{
		ans = 0;
		for(i = 0;i < strlen(a);i++)
		{
			t = a[i] - '0';
			ans = (ans * 10 + t) % b;
		}
		printf("%d\n", ans);
	}
	return 0;
}
{% endhighlight %}
