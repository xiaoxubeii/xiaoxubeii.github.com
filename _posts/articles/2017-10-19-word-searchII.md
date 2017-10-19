---
layout: post
title: "Leetcode-212 Word Search II"
description: ""
category: Algorithm
tags: [Leetcode, Algorithm]
---
Given a 2D board and a list of words from the dictionary, find all words in the board.

Each word must be constructed from letters of sequentially adjacent cell, where "adjacent" cells are those horizontally or vertically neighboring. The same letter cell may not be used more than once in a word.

For example,
Given words = `["oath","pea","eat","rain"]` and board =


```
[
  ['o','a','a','n'],
  ['e','t','a','e'],
  ['i','h','k','r'],
  ['i','f','l','v']
]
```
Return ["eat","oath"].

之前有一篇文章[Leetcode-79 Word Search](https://xiaoxubeii.github.io/algorithm/word-search/)介绍过如何在一个二维数组中找到符合要求的字符串，这题是它的变种，唯一的区别是查询的是一个字符串列表。直观的想法是循环字符串数组，然后使用DFS查询，但是这样显然无法通过时间复杂度的要求。那么在这里可以引入前缀树（字典树），将查询思路改为`以word为主体-》以board为主体`。

前缀树（字典树）的wiki定义如下：
> In computer science, a trie, also called digital tree and sometimes radix tree or prefix tree (as they can be searched by prefixes), is a kind of search tree—an ordered tree data structure that is used to store a dynamic set or associative array where the keys are usually strings. 

![400px-Trie_example.svg](/images/400px-Trie_example.svg.png)

通过使用遍历Trie判断字符串是否符合要求，然后再通过DFS查询。
以下是我编写的Trie的Python实现：

```
class TrieNode(object):
    def __init__(self):
        self.children_node = {}
        self.item = ''


class Trie(object):
    def __init__(self):
        self.root = TrieNode()

    # 在树中插入字符串
    def insert(self, word):
        node = self.root
        for c in word:
            if c not in node.children_node:
                node.children_node[c] = TrieNode()
            node = node.children_node[c]

        node.item = word

    # 查询树中是否存在字符串
    def search(self, word):
        node = self.root
        for w in word:
            if w not in node.children_node:
                return False
            node = node.children_node[w]
        return node.item == word

    # 查询树中是否有字符串的前缀
    def starts_with(self, word):
        node = self.root
        for w in word:
            if w not in node.children_node:
                return False
            node = node.children_node[w]

        return True
```

以下是我编写的该题的Python实现：


```
class Solution(object):

        def findWords(self, board, words):
            result = []
            trie_node = Trie()
            # 构造前缀树
            for w in words:
                trie_node.insert(w)

            if not board and not board[0]:
                return

            exist_matrix = []
            for i in board:
                exist_matrix.append([False] * len(i))

            def find_word(row, col, s):
                if (row < 0 or row >= len(board)) or (col < 0 or col >= len(board[row])) or exist_matrix[row][col]:
                    return
            
                s += board[row][col]
                # 前缀查询字符串
                if not trie_node.starts_with(s):
                    return
                
                # 判断字符串是否在前缀树中
                if trie_node.search(s):
                    result.append(s)
                
                exist_matrix[row][col] = True
                find_word(row - 1, col, s)
                find_word(row + 1, col, s)
                find_word(row, col - 1, s)
                find_word(row, col + 1, s)
                exist_matrix[row][col] = False

            for i in xrange(len(board)):
                for j in xrange(len(board[i])):
                    find_word(i, j, "")

            return list(set(result))
```

以上代码虽可通过Leetcode OJ，但是它的时间复杂度也很低，读者有兴趣可以自行优化。
