---
title: 关于javascript的深浅拷贝
date: 2021-08-29 16:56:13
category: Javascript
tags:
- Javascript
---
虽然照着书本了解过深拷贝和浅拷贝的大概概念，但只是有听没有懂。直到今天在bigfrontend上做题的时候才因为这件事翻了个大跟头。

<!--more-->

题目如下：

>162. 请找到未重复出现的整数
>给定一个整数数组，除了一个数字之外，其余的数字均出现了两次。请找出这个只出现了一次的数。

```
const arr = [10, 2, 2 , 1, 0, 0, 10]
findSingle(arr) // 1
```

第一次读到题目的时候觉得这题真是轻松单纯，直接套两次循环不就解决了么——这么想着敲出了这样的答案

```
/**
 * @param {number[]} arr
 * @returns number
 */
function findSingle(arr) {
  for(var i=0;i<arr.length;i++){
    tag = false
    temp = arr
    arrs = arr.splice(i,1)
    arr = temp
    for(var c=0;c<arrs.length;c++){
      if ( arrs[c] == arr[i] ){
        
        tag = true
      } 
    }
    if (tag == false){
      return arr[i]
    }
  }
}
```

然而一跑起来就傻眼了，不管把什么数组丢进去都只是把第一个数给丢出来，后来注意到以为是`arrs = arr.splice(i,1)`的问题，可是改成`arr.splice(i,1);arrs = arr`之后结果干脆全都变成了`undefined`……于是痛定思痛插了一堆`console.log`，终于在temp的取值上发现了端倪：在执行完`arr.splice(i,1)`之后，temp中的数字也会一起被移除 根本发挥不了temp的作用。

于是我去翻阅了关于JavaScript关于拷贝的文档，[segmentfault上的一篇文章](https://segmentfault.com/a/1190000018903274)是这么解释的：
> Javascript 的对象只是指向内存中某个位置的指针。这些指针是可变的，也就是说，它们可以重新被赋值。所以仅仅复制这个指针，其结果是有两个指针指向内存中的同一个地址。

也就是说，Javascript和其他的传统编程语言不同。如果不做一些特殊操作的话，`temp=arr`的效果只是将这两者指向同一个数组，并不能像我预想的那样达成拷贝一份备份的效果。我根据这点添加了`Array.from()`作为拷贝用的函数，果然程序一下子就通过了。

```
/**
 * @param {number[]} arr
 * @returns number
 */
function findSingle(arr) {
  for(var i=0;i<arr.length;i++){
    tag = false
    temp = Array.from(arr)
    arr.splice(i,1)
    for(var c=0;c<arr.length;c++){
      if ( arr[c] == temp[i] ){
        
        tag = true
      } 
    }
    if (tag == false){
      return temp[i]
    }
    arr = Array.from(temp)
  }
}
```

但是事实上在再翻阅相关资料的时候，我发现我的理解又出现了偏差：在我看来，既然能让temp与arr呈现出不同的数据，`Array.from()`应该就是做到了深拷贝，但翻阅了不少资料后发现他们都将其称为浅拷贝。直到仔细阅读完[这篇blog](https://blog.csdn.net/weixin_43513495/article/details/97934484)之后，我才找到了答案。当只有一层数组的时候，`Array.from`可以近似看作能发挥深拷贝的效果，但实际上它只是拷贝了一份指针名称组成的数组而并没有拷贝这些指针所指向的函数。打个比方的话，在通过array.from拷贝来的数组上进行操作就像是在二楼的地板上铺海绵垫，无法修改一楼向上支撑的立柱的位置。这样的拷贝是与`=`拷贝的数组近似的浅拷贝，只有将数组内包括指针指向的所有内容都拷贝的才算做深拷贝（比如无敌的`JSON.parse(JSON.stringify())`）。

具体深拷贝和浅拷贝的区别和具体的方法在引用的两篇文章里都有更深刻的讲述，如果以后我忘了就点开看看好了（发懒）