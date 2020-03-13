---
layout:     post
title:      "Recursion"
subtitle:   ""
date:       2020-01-15
author:     "Deer"
header-img: "img/star-universe-space-all-clouds-cumulus-clouds.jpg"
tags:
    - 数据结构与算法
---

> This document is not completed and will be updated anytime.

第一篇博客就交给递归大哥吧，毕竟递归对于我来说，真的好难，有的时候看不懂，有的时候看得懂但又不会写......
先阐述一下递归如何使用，然后再与letcode上的题相结合进行使用

## Catagory

1. [理论](#理论)
	1. [什么是递归](#什么是递归)
	2. [递归的三要素](#递归的三要素)
2. [实际](#实际)
	1. [题目描述](#题目描述)
	2. [解决思路](#解决思路)
3. [a-b](#a-b)

---

## 理论
### 什么是递归
> 程序调用自身的编程技巧称为递归 （百度百科）

详细来说，递归就是在有限次数的情况下，一层一层的调用自己，每次调用就是将此时的所有程序数据保存在栈中（为的是“保护现场”），以此来保证里层的程序执行完后可以安全正确返回到它的上一层程序。

### 递归的三要素
- 程序要解决什么问题
- 递归关系式
- 递归的结束条件

递归程序一般要比普通程序的理解难度大，如果再云里雾里没弄清要解决什么问题，写出正确的递归程序的可能性大大降低，因此要理解问题。
在理解问题的基础上，要推导出递推关系式，也就是怎么把当前问题的规模一步一步缩小，通过解决小问题，进而将小问题组合起来解决程序问题。
程序不可无限循环，递归也是如此，递归程序的执行次数也是有限次数。那么什么时候退出递归程序，这就要确定递归的结束条件。

---

## 实际
在理论描述完递归之后，采用letcode中的一道题目来试一把。（letcode 39.组合总和）
### 题目描述

![题目描述](/img/recursion/recursion-1.png "题目描述")
### 解决思路
- 程序要解决什么问题：
从一个无序数组中挑选若干个元素相加的和等于target.那么我们可以先对数组排序，然后依次选择，关键在于数组中的元素可以重复选择，在考虑递归关系式的时候要考虑到这个关键点
- 递归关系式
在这个问题中，需要从数组中选择若干个元素，那么关系式中包含的参数首先要有**数组(candidates)**和**数组开始位置的下标(start)**，start不可能一直为0，因为要遍历数组来选择出合适的元素。
其次还要有**target**，最后要有一个存储**当前已选择元素的集合(stack)**，这里用Stack栈来存储，因为要回溯。那么我们可以尝试将关系式定义为
find(int[] candidates,int start,int target,Stack<Integer> stack);
- 递归的结束条件
在这个递归程序中，当target为0的时候，那么就不需要再去选择元素，即程序结束。因为结束条件是target==0

完整的写出这个递归程序

 ```
  // 用来存储满足条件的数组元素集合
  private List<List<Integer>> res = new ArrayList<List<Integer>>();
  private List<List<Integer>> combinationSum(int[] candidates, int target){
	  if(candidates==null || candidates.length==0){
		  return res;
	  }
	  Arrays.sort(candidates);
	  // 递归关系式
	  find(candidates,0,target,new Stack<Integer>());
	  return res;
  }
  private void find(int[] candidates,int start,int target,Stack<Integer> stack){
	  //递归结束条件
	  if(target==0){
		  res.add(new ArrayList<Integer>(stack));
		  return ;
	  }
	  for(int i=start;i<candidates.length&&target-candidates[i]>=0;i++){
		  stack.push(candidates[i]);
		  //再次调用递归关系式，因为满足条件，所以将这个程序“递”下去
		  // 这里start依然为i的原因是可以重复选择数组中的元素，若不可以重复选择，这里应该是i+1
		  find(candidates,i,target-candidates[i],stack);
		  // 将栈顶元素弹出，看栈中剩余元素是否可以和其他元素相组合
		  stack.pop();
	  }
  }
  
```

---

## a-b

- 递归，在不满足递归结束条件的时候，将程序“递”给下一层；若满足了递归结束条件，结束该层程序，即为“归”。
- 递归程序在某种程度上可以理解为循环，但与普通的for while循环还是有些不同的。
- 递归程序的宗旨是逐步缩小问题的规模，进而结解决问题。

我会一直更新对于递归的理解，力争用最简洁的语言描述这个复杂的小东西。




