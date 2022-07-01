---
title: 在primer主题上魔改加入gitalk评论系统
date: 2021-08-31 13:58:49
category: Blog
tags:
---
primer这个主题最后一次更新的时候，多说才刚刚关站不久，主题的制作者留下了Disqus和gitment两个备选项就对这个项目撒手不管脱离战线。而2021年的今天，gitment也成为了被放弃维护的烂尾项目的一员，disqus则是被封印在墙外难以正常使用。于是作为迟到了好几年的使用者，必须为评论系统找一份新的归宿——而顺理成章的，我挑上了以gitment继任者自居的gitalk。

<!-- more -->

## 找出gitment在原项目中的配置

![vscode的搜索结果](vscode的搜索结果.png)

多亏了vscode的全局搜索功能，我们很容易就能够定位到项目中运用了gitment的文件。其中，`_config.yml`指示了传达给gitment插件的参数，`gitment-comments.js`与`article-post.ejs`是负责将gitment插件实施到页面的html模块，`style.styl`负责引入gitment的css文件，`default.css`与`gitment.browser.js`则就是gitment插件的js与css文件。接下来，就让我们手把手的引领gitalk对gitment进行一次“鸠占鹊巢”。

## 先从html部分开始讲起

让我们先来看看`article-post.ejs`中关于引入gitment的部分：

``` javascript
<%if(theme.disqus_username){%>
  <%- partial('_widget/disqus-comments',{
      key: post.slug,
      title: post.title,
      url: config.url+url_for(post.path)
  }) %>
<%} else {%>
  <%- partial('_widget/gitment-comments',{
      key: post.slug,
      title: post.title,
      url: url_for(post.path)
  }) %>
<%}%>
```

可以简单的看出，如果你在`_config.yml`中填入了`disqus_username`，这个模块就会引导`_widget/disqus-comments`以渲染disqus评论区，否则会引导`_widget/gitment-comments`来渲染gitment评论区。在gitment已经不适用的现在，我们可以将后半部分的`gitment-comments`改为
`gitalk-comments`，并在`_widget`文件夹下创建一个名称相对应的渲染模块。
至于这个渲染模块怎么创建，我们可以参考参考`gitment-comments`的写法：

``` javascript
<div class="comments">
    <div id="container"></div>
        <%- js('js/gitment.browser') %>
         <script>
            var gitment = new Gitment({
                id: '<%=url%>',
                owner: '<%= theme.gitment_owner %>',
                repo: '<%= theme.gitment_repo %>',
                oauth: {
                    client_id: "<%= theme.gitment_client_id %>",
                    client_secret: "<%= theme.gitment_client_secret %>",
                }
            })
        gitment.render('container')
        </script>
</div>
```

同时，我们可以拿出gitalk文档中的引用方法进行比对：

``` javascript
var gitalk = new Gitalk({
  clientID: 'GitHub Application Client ID',
  clientSecret: 'GitHub Application Client Secret',
  repo: 'GitHub repo',
  owner: 'GitHub repo owner',
  admin: ['GitHub repo owner and collaborators, only these guys can initialize github issues'],
  id: location.pathname,      // Ensure uniqueness and length less than 50
  distractionFreeMode: false  // Facebook-like distraction free mode
})

gitalk.render('gitalk-container')
```

可以看出，gitalk与gitment的设计方式是一脉相承的。我们只需要依照gitment中配置好的接口将gitalk的语句改头换面进去就行了。下面展示的是修改完成之后的结果。

``` javascript
<div class="comments">
    <div id="container"></div>
        <%- js('js/gitalk') %>
        <script>
            var gitalk = new Gitalk({
                clientID: "<%= theme.gitment_client_id %>",
                clientSecret: "<%= theme.gitment_client_secret %>",
                repo: '<%= theme.gitment_repo %>',
                owner: '<%= theme.gitment_owner %>',
                admin: ['<%= theme.gitment_owner %>',],
                id: '<%=url%>',
            })
        gitalk.render('gitalk-container')
        </script>
</div>
```

## 在本地引入gitalk的css与js库

在前文里我们修改了gitment中属于html的部分。而js与css部分则更加单纯——我们只需要从gitalk的github仓库上拖下来对应的js与css文件再完成引入就行了。js的引入我们已经在上文中做了修改（`<%- js('js/gitment.browser') %>`被修改为`<%- js('js/gitalk') %>`），css的引入则位于`style.styl`文件中：

``` javascript
//gitmnet
@import "_vendor/gitment/default.css"
```

我们将原本的引入文本注释掉，重新加入gitalk的css

``` javascript
//gitment
//@import "_vendor/gitment/default.css"

//gitalk
@import "_vendor/gitalk/default.css"
```

最后将gitalk仓库中dist文件夹里的文件拖到本地我们写好的对应位置，gitalk的引入就大功告成了……吗？

![有人又写bug了_啊_我不说是谁](有人又写bug了_啊_我不说是谁.png)

笑死，我又在写bug哦。

## Bug排除

然后，我开启了漫长的查bug之旅……

首先是发现自己把`gitalk`的名字打成了`gittalk`，打开搜索工具发现出现了一百多条gittalk这个本来不该存在的名字。一键全局替换……好的没有搞定bug依旧好端端的呆在那里。

之后我又拿出原本gitment版本渲染完成的界面进行比对，发现修改成gitalk后浏览器的`container`标签内不再渲染内容而这本来应该渲染出评论框，于是我把目光投向了渲染出评论的`render`……结果很相近了，但只差一步，我倒在自己看不懂的六万多行的代码之海里找不着头绪。

就在此时我才突然想到也许应该再看看官方文档……直到这个时候我才发现，`gitalk`的评论区是渲染在`gitalk-container`这个标签里的，也就是说必须要把`container`改为`gitalk-container`才能正常运行。于是我试着再度更改了`gitment-comments`的写法……

``` javascript
<div class="comments">
    <div id="gitalk-container"></div>
        <%- js('js/gitalk') %>
        <script>
            var gitalk = new Gitalk({
                clientID: "<%= theme.gitment_client_id %>",
                clientSecret: "<%= theme.gitment_client_secret %>",
                repo: '<%= theme.gitment_repo %>',
                owner: '<%= theme.gitment_owner %>',
                admin: ['<%= theme.gitment_owner %>'],
                id: '<%=url%>',
            })
        gitalk.render('gitalk-container')
        </script>
</div>
```

好的来吧我要按下去(`hexo s`)了！

![OHHHHHHHH](OHHHHHHHH.png)

<font size=6>大 功 告 成！</font>

## 碎碎念

这次折腾完加上上一篇感觉得整合整合把自己改掉的部分丢到自己fork的那个分支上……花了这么多时间折腾上这些只是这样写成博客太可惜了。话说回来这样算不算参加开源项目？所以那些参加开源项目的人都是用这种心情去改别人的代码的吗？！（哭笑不得）

另外就是为什么在Github pages上更新文章不算在commit里，周末两天的白框框看的我心在滴血，可恶。
