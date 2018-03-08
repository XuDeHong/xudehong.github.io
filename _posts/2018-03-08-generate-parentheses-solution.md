---
layout: post
title: Generate Parentheses 解题笔记
date: 2018-03-08 11:29:24.000000000 +09:00
tags: 算法

---

原题链接：[Generate Parentheses](https://leetcode.com/problems/generate-parentheses/description/)

Given *n* pairs of parentheses, write a function to generate all combinations of well-formed parentheses.

For example, given *n* = 3, a solution set is:

```
[
  "((()))",
  "(()())",
  "(())()",
  "()(())",
  "()()()"
]
```

## 解题思路

首先，我们知道最后得出的每一个解的左括号数目和右括号数目肯定是相等并且一对一匹配的，那么任意时刻能够作出的操作就是下面三种情况：

1. 添加左括号
2. 添加右括号
3. 得到一个解

典型的递归方法，用`left`和`right`两个变量分别记录`左括号的剩余数目`和`右括号的剩余数目`。每添加一次左括号，left-1，right+1；每添加一次右括号，left不变，right-1。直到left和right相等并等于零时，等到一个有效解。这时递归返回，再递归前进，直到所有解都求出来。

```c
#define MAX_COUNT 10000 //假设最多有这么种情况

void addParenthesisRecursively(int n,int left, int right, int *returnSize, char *str, char **result) {
    if((left == 0) && (right == 0))
    {
        result[(*returnSize)++] = str;
    }
    else
    {
        char *newStr = (char *)malloc(sizeof(char)*(2*n+1));
        if(left > 0)
        {
            strcpy(newStr,str);
            addParenthesisRecursively(n,left-1,right+1,returnSize,strcat(newStr,"("),result);
        }
        if(right > 0)
        {
            strcpy(newStr,str);
            addParenthesisRecursively(n,left,right-1,returnSize,strcat(newStr,")"),result);
        }
    }
}

char** generateParenthesis(int n, int* returnSize) {
    char **result = (char **)malloc(sizeof(char *) * MAX_COUNT);
    addParenthesisRecursively(n,n,0,returnSize,"",result);
    return result;
}

int main(int argc, const char * argv[]) {
    // insert code here...
    int returnSize = 0;
    char** result = generateParenthesis(3, &returnSize);
    
    return 0;
}
```

## 算法思想

本题所用到的算法思想是`回溯法`，它在百度百科的定义如下：

>[回溯法](https://baike.baidu.com/item/%E5%9B%9E%E6%BA%AF%E6%B3%95)（探索与回溯法）是一种选优搜索法，又称为试探法，按选优条件向前搜索，以达到目标。但当探索到某一步时，发现原先选择并不优或达不到目标，就退回一步重新选择，这种走不通就退回再走的技术为回溯法，而满足回溯[条件](https://baike.baidu.com/item/%E6%9D%A1%E4%BB%B6/1783021)的某个[状态](https://baike.baidu.com/item/%E7%8A%B6%E6%80%81)的点称为“回溯点”。

以上面那道题为例，假设3对括号求解，那么第一次递推的最终结果是str=“((()))”，接着递归返回，直到str=“((”，这时满足if(right > 0)条件，开始加右括号“)”，然后是左括号“(”，以此类推，不断递推前进，得到第二个结果str="(()())"，再递归返回，不断重复这个过程，最后得出所有结果。具体过程可以用上面的代码调试看看，感受下这个回溯的过程。通俗点讲，回溯法就是能够前进的，继续前进，不能前进的则返回，如果达到目标获得一个解，则返回，去尝试求得另一个解。