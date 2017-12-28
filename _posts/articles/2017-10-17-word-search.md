---
layout: post
title: "Leetcode-79 Word Search"
description: ""
category: articles
tags: [Leetcode, Algorithm]
---
Given a 2D board and a word, find if the word exists in the grid.

The word can be constructed from letters of sequentially adjacent cell, where "adjacent" cells are those horizontally or vertically neighboring. The same letter cell may not be used more than once.

For example,
Given board =


```
[
  ['A','B','C','E'],
  ['S','F','C','S'],
  ['A','D','E','E']
]

```
word = "ABCCED", -> returns true,
word = "SEE", -> returns true,
word = "ABCB", -> returns false.

这道题是给定一个二维的数组和一个字符串，查找该字符串是否在二维数组网格内。字符串中的字符在网格内必须相邻，可以是垂直或者水平的。并且网格内字符只能使用一次。

该题可以使用DFS算法解决。具体思路是，在上下左右四个方向递归查询一个字符是否存在于网格内。在这里需要着重注意的是网格的边界。
代码如下：

```
class Solution(object):
    def exist(self, board, word):
        if not board and not board[0]:
            return False
        
        # 构造二维矩阵，第ij位表示字符是否使用
        exist_matrix=[]
        for i in board:
            exist_matrix.append([False]*len(i))

        def find_word(row,col,index):
            # 递归结束条件，字符串已经找到
            if index==len(word):
                return True

            # 判断边界条件等
            if (row < 0 or row >= len(board)) or (col < 0 or col >= len(board[row])) or exist_matrix[row][col] or board[row][col] != word[index]:
                return False

            exist_matrix[row][col] = True
            
            # 在上、下、左、右四个方向查找字符
            if find_word(row-1, col, index+1): 
                return True
            if find_word(row+1, col, index+1):
                return True
            if find_word(row, col-1, index+1):
                return True
            if find_word(row, col+1, index+1):
                return True

            exist_matrix[row][col] = False
            return False

        # 循环访问二维数组
        for i in xrange(len(board)):
            for j in xrange(len(board[i])):
                if find_word(i, j, 0):
                    return True

        return False
        
```

