---
layout: post
title: "Leetcode-200 Number of Islands"
description: ""
category: Algorithm
tags: [Leetcode, Algorithm]
---

Given a 2d grid map of '1's (land) and '0's (water), count the number of islands. An island is surrounded by water and is formed by connecting adjacent lands horizontally or vertically. You may assume all four edges of the grid are all surrounded by water.

Example 1:

```
11110
11010
11000
00000
```
Answer: 1

Example 2:

```
11000
11000
00100
00011

```

Answer: 3

这道题比较简单，是使用dfs递归。但是这里有个技巧，在找到一个‘1’也就是land时，需要将该位及其四周都置成‘0’，代表找到了一个island。

我实现的代码如下：

```
class Solution(object):
    def numIslands(self, grid):
        """
        :type grid: List[List[str]]
        :rtype: int
        """
        result=0
        if not grid or not grid[0]:
             return result
            
        # DFS递归
        def dfs(row,col):
            if row<0 or row>=len(grid) or col<0 or col>=len(grid[row]) or grid[row][col]!='1':
                return
            
            # 找到‘1’后将该位置成‘0’
            grid[row][col]='0'
            dfs(row-1,col)
            dfs(row+1,col)
            dfs(row,col-1)
            dfs(row,col+1)
            
        for i in xrange(len(grid)):
            for j in xrange(len(grid[i])):
                # 寻找island
                if grid[i][j]=='1':
                    dfs(i,j)
                    result+=1
                    
        return result
```
