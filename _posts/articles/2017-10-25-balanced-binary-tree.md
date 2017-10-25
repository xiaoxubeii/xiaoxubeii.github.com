---
layout: post
title: "Leetcode-110 Balanced Binary Tree"
description: ""
category: Algorithm
tags: [Leetcode, Algorithm]
---
Given a binary tree, determine if it is height-balanced.

For this problem, a height-balanced binary tree is defined as a binary tree in which the depth of the two subtrees of every node never differ by more than 1.

此题是判断是否为高度平衡二叉树，所谓高度平衡二叉树就是左右子树的高度差不超过1。可以通过深度递归求得树的高度，然后判断是否为高度平衡。


```
# Definition for a binary tree node.
class TreeNode(object):
    def __init__(self, x):
        self.val = x
        self.left = None
        self.right = None

class Solution(object):
    def isBalanced(self, root):
        """
        :type root: TreeNode
        :rtype: bool
        """
        
        # dfs递归求树的深度
        def get_depth(root):
            if not root:
                return 0

            return 1 + max(get_depth(root.left),get_depth(root.right))
        
        if not root:
            return True
        # 判断是否为高度平衡二叉树
        if abs(get_depth(root.left) - get_depth(root.right)) > 1:
            return False
        
        return self.isBalanced(root.left) and self.isBalanced(root.right)
        
```

显然，以上深度遍历了每个节点，这是不必要的。我们可以在遍历子树的时候就判断子树是否为平衡二叉树，然后不是的话就直接返回-1，如果是的话就返回子树深度。

所以代码可修改为：

```
# Definition for a binary tree node.
class TreeNode(object):
    def __init__(self, x):
        self.val = x
        self.left = None
        self.right = None
        
class Solution(object):
    def isBalanced(self, root):
        """
        :type root: TreeNode
        :rtype: bool
        """
        def check_depth(root):
            if not root:
                return 0
            
            left=check_depth(root.left)
            if left==-1:
                 return -1
                
            right=check_depth(root.right)
            if right==-1:
                return -1
            
            if abs(left-right)>1:
                return -1
            
            else:
                return 1+max(left,right)
            
        return check_depth(root)!=-1
            
```



