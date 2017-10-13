---
layout: post
title: "Leetcode-567 Permutation in String"
description: ""
category: Algorithm
tags: [Leetcode, Algorithm]
---

今天在刷Leetcode算法题时，遇到一道求字符排列的问题：

```
Given two strings s1 and s2, write a function to return true if s2 contains the permutation of s1. In other words, one of the first string's permutations is the substring of the second string.
Example 1:
Input:s1 = "ab" s2 = "eidbaooo"
Output:True
Explanation: s2 contains one permutation of s1 ("ba").
Example 2:
Input:s1= "ab" s2 = "eidboaoo"
Output: False
Note:
The input strings only contain lower case letters.
The length of both given strings is in range [1, 10,000].
```

也就是说给定两个字符串s1和s2，编写一个函数，实现s2若包含s1的字符排列就返回true。
拿到问题，直观的想法是使用穷举将s1中所有可能的字符排列全部找到，然后再在s2中查找。但是s1和s2的长度范围在1-10000之间，穷举的话算法的时间复杂度肯定无法满足要求。那么，可以使用Sliding Window的思想来解决问题。

使用两个哈希表，记s1的长度为n1，分别统计n1个长度内s1和s2中各字符出现的次数，如果相等，则返回true，这时说明s2中有s1的排列。如果不相等，依次统计n1后s2中各字符出现的次数，但这时指针每往右移动一位，就需要将首位的字符移除（实际是出现次数减1），然后比较两个哈希表是否相等。可以将这个过程想象成一个不断向右移动的固定长度的窗口，然后比较两个窗口的统计数据是否一致。

我实现的Python版代码如下：


```
class Solution(object):
    def checkInclusion(self, s1, s2):
        a={}
        b={}
        n1=len(s1)
        n2=len(s2)
        
        if n1>n2:
            return False
        
        # 统计前n1个字符出现次数
        for i in xrange(n1):
            a[s1[i]]= 1 if s1[i] not in a else a[s1[i]]+1
            b[s2[i]]= 1 if s2[i] not in b else b[s2[i]]+1

        # 相等返回true
        if a==b:
            return True

        for i in xrange(n1,n2):
            # 统计n1后s2中字符出现的次数，指针右移一位
            b[s2[i]]= 1 if s2[i] not in b else b[s2[i]]+1

            # 移除首位字母，出现次数减1，相当于窗口右移一位
            if b[s2[i-n1]]==1:
                b.pop(s2[i-n1])
            else:
                b[s2[i-n1]]-=1

            if a==b:
                return True

        return False

```
