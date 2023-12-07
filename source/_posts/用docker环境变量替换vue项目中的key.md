---
title: 用docker环境变量替换vue项目中的key
date: 2023-12-07 09:40:47
category: Javascript
tags: Javascript
---

因为用的实在太多了又老是忘网上又神秘的不好找所以自己写一篇备份！

<!--more-->

### 在Dockerfile中使用envsubst将环境变量传入index.html

``` bash
FROM nginx:mainline
COPY dist /usr/share/nginx/html
WORKDIR /etc/nginx/conf.d
COPY ./nginx.conf /etc/nginx/nginx.conf.tmpl
EXPOSE 80
CMD /bin/sh -c "envsubst < /etc/nginx/nginx.conf.tmpl > /etc/nginx/nginx.conf \
# ***修改的重点，将变量置换进index.html***
&& sed -i \"s|<body>|<body \
  SECRET_KEY=\"$SECRET_KEY\" \
>|\" /usr/share/nginx/html/index.html \
# ***这里是重点***
&& nginx -g 'daemon off;' || cat /etc/nginx/nginx.conf"
```

### 在public/index.html中接收环境变量存入window域

``` html
  <body>
    ……
    <!-- 添加script标签，将环境变量接受进window域 -->
    <script type="text/javascript">
      var secret_key = document.querySelector('body').getAttribute('SECRET_KEY');
      document.querySelector('body').removeAttribute('SECRET_KEY');
    </script>
  </body>
```

### 在具体页面中调取windows域中的环境变量

``` javascript
  mounted() {
    // 调取环境变量
    this.secret_key = window.secret_key;
  },
```
