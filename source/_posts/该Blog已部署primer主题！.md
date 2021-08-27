---
title: 该Blog已部署Primer主题！
date: 2021-08-26 12:40:37
category: Blog
tags:
- Blog
---
虽然没有这样的常言，但常言道部署完hexo的第一件事就应该是找主题。在hexo的官方文档里挑花了眼也不知道该选什么主题比较好，在一阵走马观花的挑选之后我最终决定了既简洁时尚又功能繁多的Fluid————本来应该是这样的，然而我在最后的最后被一篇介绍Fluid的博客的主题给截胡，一眼相中了简介大气重点突出的Primer并义无反顾地火速配置了主题。

相比fluid，Primer是个年久失修的老旧主题，最后一次更新远在2017年，彼时hexo的版本还停留在3.0，如今光是版本号的数字就已经翻了近一倍（截至2021年8月26日，hexo的最新版本号为5.4.0）。但老归老，鼓捣鼓捣还能用。也顺手在这篇文章里写写配置Primer时遇到的坑，如果有谁和我一样为了这些坑整的焦头烂额的时候搜到了这篇文章能帮上忙那就太好了。

<!--more-->

### 部署到Github Pages时主题失效问题

因为Hexo 5.0.0以前的版本不支持直接安装主题，Primer的文档里采用的是将主程序`git clone`到本地再在`_config.yml`中手动引导theme的方法。这个方法在本地使用的时候没什么问题——当然前提是记得要先cd到themes目录下再`git clone`——但是在推送到Github Pages的时候就会发现themes文件夹无法和其他文件一并上传，并且会导致Blog主页失去所有样式。但不论是检查gitignore还是执行`hexo clean`都无法将themes推送上去。

而这个问题的解决方法其实出人意料的单纯和简单。因为主题是通过`git clone`的方式搭载的，而`hexo d`在检测到这是属于他人的git仓库之后就会放弃将其push到远程仓库。所以只要删除hexo-theme-Primer文件夹下的.git文件夹就能一劳永逸的解决问题了。<del>而且毕竟这也是个没人维护的古老项目了也不用担心后续更新什么的</del>

### footer的署名与RSS问题

Primer在footer处默认的署名是`© 2016 yumemor`并且没有给出更改的接口，但既然是自己的Blog，想要在footer上署上自己的名字也是人之常情。幸亏Primer的结构非常单纯，在`hexo-theme-primer/layout/_partial/footer.ejs`文件中，我们可以轻松更改footer的写法。另外，因为这个Blog目前并没有配置RSS，也可以在这里将RSS入口删去。

在这里贴出我参考[春上冰月](https://github.com/haruue)的Blog修改过的`footer.ejs`段落：

```
<footer class="container">
    <div class="site-footer" role="contentinfo">
        <div class="copyright left mobile-block">
                © 2021
                <span title="yumemor">SomiaWhiteRing</span>
                <a href="javascript:window.scrollTo(0,0)" class="right mobile-visible">TOP</a>
        </div>

        <ul class="site-footer-links right mobile-hidden">
            <li>
                <a href="javascript:window.scrollTo(0,0)" >TOP</a>
            </li>
        </ul>

        <a href="https://github.com/SomiaWhiteRing" target="_blank" aria-label="view source code">
            <span class="mega-octicon octicon-mark-github" title="GitHub"></span>
        </a>

        <ul class="site-footer-links mobile-hidden">
            <% for (var nav in theme.menus) { %>
                  <%
                    var title = theme.menus[nav].name;
                    var href = theme.menus[nav].link;
                    var link = "^" + href;  
                    var target = theme.menus[nav].target;
                  %>
                  <li>
                    <a href="<%- url_for(href) %>" <%if(target){%>target="<%=target%>"<%}%> title="<%= title %>"><%= title %></a>
                  </li>
            <% } %>
        </ul>

        <div>
            <span title="hexo">由 <a target="_blank" rel="noopener" href="https://hexo.io/">hexo</a> 驱动</span>
            <span title="yumemor">并使用基于 <a target="_blank" rel="noopener" href="https://github.com/yumemor/hexo-theme-primer">primer</a> 的主题</span><br>
        </div>
        
    </div>
</footer>

```

好的到目前为止遇到的问题都说了个大概，如果接下来又遇到什么问题的话大概也会更新在这篇文章……不得不说虽然不希望出bug但是 嘛 这种年久失修的项目不出点问题反而才不正常对吧（