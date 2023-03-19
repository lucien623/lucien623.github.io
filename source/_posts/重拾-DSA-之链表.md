---
title: 重拾  DSA 之链表
date: 2017-08-11 15:40:59
tags: 
  - DSA
categories:
  - 技术
description: 本文主要介绍了链表的一些操作。
---
下面是 wikipedia 对于链表的定义：

> **链表**（Linked list）是一种常见的基础数据结构，是一种线性表，但是并不会按线性的顺序存储数据，而是在每一个节点里存到下一个节点的指针(Pointer)。由于不必须按顺序存储，链表在插入的时候可以达到   O(1) 的复杂度，比另一种线性表顺序表快得多，但是查找一个节点或者访问特定编号的节点则需要 O(n) 的时间，而顺序表相应的时间复杂度分别是 O(logn) 和 O(1)。

- 添加
  ```java
  private void add(Object data) {
		//如果头节点为空，则创建一个新节点，并将其设置为头节点以及当前节点。
		if(head == null) {
			head = new Node(data);
			current = head;
		}
		else {
			current.next = new Node(data);
			current = current.next;
		}
	}
  ```
  
  - 插入
  ```java
  private void insert(int index, Object data) {
		current = head;
		if(index == 0) {
			head = new Node(data);
			head.next = current;
			return;
		}
		Node preNode = null;
		while(current != null) {
			if((int)current.data == index) {
				Node node = new Node(data);
				preNode.next = node;
				node.next = current;
				return;
			}
			preNode = current;
			current = current.next;
		}
	}
  ```
  
  - 删除
  ```java
  private void delete(int index) {
		current = head;
		int pos = 0;
		Node preNode = head;
		while(current != null) {
			if(pos == index) {
				if(pos == 0)
					head = current.next;
				//判断当前节点是否还有下一个节点，有则将前一个节点的next指向下一个节点，
				//无则将前一个节点的next指向设置为null
				else {
					if(current.next != null) {
						preNode.next = current.next;
					}
					else {
						preNode.next = null;
					}
				}
				return;
			}
			preNode = current;
			current = current.next;
			pos++;
		}
	}
	```
	
  - 查找
  ```java
  private Node find(int index) {
		current = head;
		int pos = 0;
		while(current != null) {
			if(index == pos) {
				return current;
			}
			current = current.next;
			pos++;
		}
		return null;
	}
	```
	
  - 获取链表中节点的个数
  ```java
  private int getLinkListLength() {
		current = head;
		int length = 0;
		while(current != null) {
			length++;
			current = current.next;
		}
		return length;
	}
  ```
  
  - 链表反转
  ```java
 private void reverseLinkList() {
		current = head;
		while(current != null) {
			Node preHead = reverseHead;
			reverseHead = current;
			current = current.next;
			if(preHead != null) {
				reverseHead.next = preHead;
			}
			else {
				reverseHead.next = null;
			}
		}
	}
  ```
  
  - 链表反转（递归方式）
  ```java
  private Node reverseLinkListRec(Node head) {
		if(head == null || head.next == null)
			return head;
		Node newHead = reverseLinkListRec(head.next);
		head.next.next = head;
		head.next = null;
		return newHead;
	}
  ```
  
  - 获取倒数第 index 节点（先让当前节点移动到第 index 个节点，再创建一个节点 start 指向头节点，当前节点和 start 节点同时移动，当当前节点移动到最后一个节点时，start 指向的便是倒数第 index 节点）
  ```java
 private Node getReciprocalNode(int index) {
		if(index == 0)
			return null;
		current = head;
		Node start = head;
		while(current != null) {
			index--;
			if(index == 0)
				break;
			current = current.next;
		}
		// 长度大于链表长度
		if(current == null) {
			return null;
		}
		while(current.next != null) {
			current = current.next;
			start = start.next;
		}
		return start;
	}
  ```

  - 获取链表的中间节点(初始化两个节点都指向头节点，一个节点每次移动一个节点，另一个每次移动两个节点，当第二个节点移动到链表末尾时，第一个节点指向的节点即是链表的中间节点)
  ```java
  private Node getMiddleNode() {
		Node oneStepNode = head;
		Node twoStepNode = head;
		//加入小于三个节点则返回第一个节点作为中间节点
		if(head.next == null || head.next.next == null) {
			return current;
		}
		while(twoStepNode.next != null && twoStepNode.next.next != null) {
			twoStepNode = twoStepNode.next.next;
			oneStepNode = oneStepNode.next;
		}
		return oneStepNode;
	}
  ```
  
[Source Code](https://github.com/lucien623/DSA_Review/blob/master/LinkList.java)


