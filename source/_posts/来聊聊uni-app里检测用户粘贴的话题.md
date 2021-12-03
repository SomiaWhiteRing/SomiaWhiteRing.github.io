---
title: 来聊聊uni-app里检测用户粘贴的话题
date: 2021-11-10 19:40:38
category: uni-app
tags:
---

前言暂略，总之今天苍旻一如既往地和issues搏斗的时候，收到了这么一个需求：

![密码框中未限制不可粘贴输入.png](密码框中未限制不可粘贴输入.png)

想着这有什么的难的的苍旻随手按了个确定，然后就寻找起了解决办法——然而他这时还没想到的是，他手头的工具不是延展性高到无所不能的原生HTML三件套；也不是虽然API压得乱七八糟但好歹有无数人陪着一同戴着镣铐跳舞的微信wxml；而是套了两层三层娃，为了兼容性在两者的语法中撕开一条血路的uni-app。

<!--more-->

## 原生写法——什么的果然行不通

`onpaste` 是个相当老牌的检测粘贴事件了。虽然不在W3C标准里，但从IE8开始的所有浏览器都有提供对它的支持。它的效果也正如其名，当指定的标签中发生粘贴事件时触发对应的函数——也许比起 `onpaste`，人们对他的孪生兄弟 `oncopy` 会更加熟悉。找了半天终于找到想要的资料准备选中复制，却在内容被收进剪贴板的前一刻被唐突打断进而跳出诱导关注微信公众号或者付费的窗口，往往就是这个事件的杰作。

但是理所当然的，这种旮沓角落里的冷门事件入不了微信的法眼。哪怕在最外层的view里放进这个事件，也不管怎么按下粘贴都不会有任何反应——甚至连个报错都没有就直接被当成代码中的杂质无视掉了。

## 啊等等，小程序这不是有自己的解法吗？

因为实际开发中需要用到这个事件的场景意外的多，微信开放社区里时不时就会冒出相关的吐槽。经过多年零零星星的讨论，社区里的人逐渐形成一项共识：使用微信为 `input` 组件提供的 `bindinput` 能力，通过打迂回战的方式来解决问题。

这场迂回战究竟要怎么打呢？这就不得不来讲讲 `bindinput` 的能力了： `bindinput` 能够实时检测小程序中最新的输入，并以 `{value, cursor, keyCode}` 的格式返回。所谓的迂回战法，就是每次输入都检测一次这个 `value` 值的长度，倘若 `value.length > 1` 就代表用户并不是通过键盘正常输入——也就可以认为是使用了粘贴键。而在这时立刻删去输入框中与 `value` 等长的字符，就能做到和禁止粘贴完全相同的效果。

看起来简直是天衣无缝的计划——事实上也毫无漏洞，毕竟使用这个事件的场景基本不是密码就是银行卡号，唯一的漏洞如邮箱后缀之类的定型文也基本都在事件的狩猎范围之外。而在我兴致勃勃的想要用上这一招的时候，开发者工具的报错给了我当头一棒：

```nohighlight
Component "pages/login/login" does not have a method "testPaste" to handle event "input".
```

## 结果说到底还是得靠自己

简单的讲，`uni-app` 的开发基础是 `Vue` 的三段式模板语法。`html`,`css`,`javascript`各司其职，被包裹在各自的模块之中。而微信小程序里，专有能力触发的事件会安置在 `Component` 构造器中——uni-app理所当然的没有这个 `Component` 构造器，想要使用 `bindinput` 自然也就变成了无稽之谈。

不过，这一条走不通的路却给了我灵感。虽然没法检测用户的输入，但检测输入框的变动却是vue的拿手好戏。只要拿变动后的字符长度与变动前的进行比对，不就能判断出用户是否使用了粘贴这一招吗？

手随心动。我马不停蹄地把想法化作现实：

```html
<template>
  ……
  <u-field
    placeholder="请输入密码"
    v-model="password"
    @input='testPaste'
  />
  ……
</template>
```

```javascript
<script> 
  export default{
    data(){
      return{
        ……
        password：'',
        testPasteVal:'',
      }
    },
    methods:{
      ……
      //测试密码是否粘贴
      testPaste(e){
        let newlength = e.length - this.testPasteVal.length
        if(newlength > 1){
          console.log("使用了粘贴!")
        }
        this.testPasteVal = e
      }
      ……
    }
  }
</script>
```

随后我便将这串代码拿到了测试上——效果相当令人满意。可比起 `bindinput` 法还是有一个缺陷：如果用户提前选中了原本的文字，那么如果粘贴的文本长度比原本的要短，就没法检测到问题所在了。

但既然有题也就有解。LeetCode上就有这么道名为`Edit Distence`的题目来探讨两个字符串的最短编辑距离，而无数的做题家则已经把这道题解了个底朝天。正常的输入/删除操作只会有1的编辑距离，我们可以将这道题的解化用在这段程序上：如果检测到编辑距离大于1，则将用户输入前的值重新赋给 `password` ：

```javascript
  ……
  //测试密码是否粘贴
    testPaste(e){
    let newlength = e.length - this.testPasteVal.length
    let distance = this.minDistance(this.testPasteVal,e)
    console.log()
    if(newlength > 1 || distance > 1){
      console.log("使用了粘贴!")
      this.password = this.testPasteVal
    }else{
      this.testPasteVal = this.password
    }
    console.log(this.password)
  },
  //计算编辑距离
    minDistance(str1, str2) {
    const length1 = str1.length;
    const length2 = str2.length;

    if(!length1) {
      return length2;
    }

    if(!length2) {
      return length1;
    }

    // i 为行，表示str1
    // j 为列，表示str2
    const r = [];
    for(let i = 0; i < length1 + 1; i++) {
      r.push([]);
      for(let j = 0; j < length2 + 1; j++) {
        if(i === 0) {
          r[i][j] = j;
        } else if (j === 0) {
          r[i][j] = i;
        } else if(str1[i - 1] === str2[j - 1]){
          r[i][j] = r[i - 1][j - 1];
        } else {
          r[i][j] = Math.min(r[i - 1][j ], r[i][j - 1], r[i - 1][j - 1]) + 1;
        }
      }
    }

    return r[length1][length2];
  },
  ……
```

——看起来已经万无一失了，但是放进实机里却只能管住data里的数据而管不到前台的输入。找不着头绪的我，偶然在[这篇文章](https://www.cnblogs.com/sinosaurus/p/10963526.html)里找到了问题所在：我使用的组件并不是原生的`input` 而是uview再封装的 `u-field` ，为了将数据的变化暴露出来，必须使用 `$nextTick`进行额外处理。

修改后的代码成了这样：

```javascript
  ……
  //测试密码是否粘贴
    testPaste(e){
    //既然采用了距离法，就没有必要做length比较了
    let distance = this.minDistance(this.testPasteVal,e)
    if(distance > 1){
      console.log("使用了粘贴!")
      this.$nextTick(() => {
        this.password = this.testPasteVal
      })
    }else{
      this.testPasteVal = this.password
    }
  },
  ……
```

把这个版本的代码丢进测试——大功告成！苍旻再次用自己~~谷歌~~的力量下uni-app了一城！
