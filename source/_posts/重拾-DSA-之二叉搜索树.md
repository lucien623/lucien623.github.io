---
title: 重拾 DSA 之二叉搜索树
date: 2019-10-26 13:49:48
tags: 
  - DSA
categories:
  - 技术
description: 本文主要介绍了二叉搜索树的基本知识。
---

二叉搜索树即 Binary Search Tree，也可以称为二叉查找树或者有序二叉树。它可以是空树或者是具有以下性质的二叉树（from wiki）：
> 1.若任意节点的左子树不空，则左子树上所有节点的值均小于它的根节点的值；
2.若任意节点的右子树不空，则右子树上所有节点的值均大于它的根节点的值；
3.任意节点的左、右子树也分别为二叉查找树；
4.没有键值相等的节点。

首先定义节点
```java
class TreeNode {
	TreeNode left,right;
	int value, num = 1;
	public TreeNode(int value) {
		this.value = value;
	}
}
```
插入节点，如果根节点为空则直接插入，否则与当前节点的值判断，如果大于当前节点，则和左子树中的节点判断，反之则和右子树中的节点做判断，直到比较到叶子节点最终确定插入位置，相等的特殊情况则将当前节点的 num 加 1，结束插入节点
```java
private void insertNode(TreeNode node) {
	if(root == null) {
		root = node;
		return;
	}
	TreeNode temp = root;
	for(;;) {
		if(node.value > temp.value) {
			if(temp.right != null)
				temp = temp.right;
			else {
				temp.right = node;
				return;
			}
		}
		else if (node.value < temp.value) {
			if(temp.left != null)
				temp = temp.left;
			else {
				temp.left = node;
				return;
			}
		}
		else {
			temp.num++;
			return;
		}
	}
}
```
删除节点的逻辑分三种情况，如果待删除的节点是叶子节点，那么可以直接删除；如果待删除的节点只有左子树或者右子树，那么可以重接左子树或右子树，即把待删除节点的左节点或者右节点直接替代；如果待删除节点既有左子树又有右子树，那么把左子树中的最小值替代待删除节点。
```java
private boolean deleteNode(int value) {
	if(root == null)
		return false;
	TreeNode parent;
	TreeNode current = parent = root;
	for(;;){
		if(current.value == value)
			break;
		parent = current;
		if(current.value < value)
			current = current.right;
		else 
			current = current.left;
		if(current == null)
			return false;
	}
	//叶子节点，直接删除
	if(current.left == null && current.right == null) {
		 actualDelete(parent, current, null);
	}
	//左子树为空,重接右子树
	else if(current.left == null){
		actualDelete(parent, current, current.right);
	}
	//右子树为空,重接左子树
	else if(current.right == null){
		actualDelete(parent, current, current.left);
	}
	//左子树和右子树都不为空，那么将右子树中的最小节点替代被删除的节点
	else {
		TreeNode temp = current.left;
		TreeNode tempParent = current;
		while(temp.left != null) {
			tempParent = temp;
			temp = temp.left;
		}
		actualDelete(parent, current, temp);
		temp.left = current.left;
		temp.right = current.right;
		tempParent.left = null;
	}
	
	return true;
}

private void actualDelete(TreeNode parent, TreeNode deleteNode, TreeNode next) {
	if(parent.left == deleteNode)
		parent.left = next;
	else 
		parent.right = next;
}
```
可以用中序遍历输出有序序列
```java
private void inOrder(TreeNode node) {
	if(node != null) {
		inOrder(node.left);
		for(int i = 0;i < node.num; i++)
			System.out.print(node.value + "-->");
		inOrder(node.right);
	}
}
```
二叉搜索树查找、插入的期望复杂度是 O(log n)，最坏情况下即退化成线性表的时候为 O(n)。