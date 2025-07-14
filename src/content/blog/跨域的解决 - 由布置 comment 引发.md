---
title: "跨域的解决 - 在博客里面添加一个评论系统"
publishDate: "2025-06-28"
description: "花了点时间配了下 waline 评论系统，作为第一篇技术博客啦"
tags: ["玩具项目"]
---

# 这是一个一级标题

## 这是一个二级标题

> 因为这个主题没有支持跨级标题，因为标题是树状结构的，只能有一个根，而且孩子的 depth 只能比 parent 深一层。个人的写作习惯又是从三级标题开始（这个习惯是有缘由的，挖个坑以后说）。所以可能后面我的每篇博客都会有这俩玩意。

### 没能免俗

看过不少同行的博客，发现好多博客的第一篇技术博客都是“用 xx 分钟搭一个属于你的个人博客”。哈，果然我也没能“免俗”（其实一点也不俗好嘛，都不写我怎么学）。

但是现在这个 astro + vercel + react 的工具链确实非常好用，找到个这个挺好看的 theme 之后基本是 fork 下来随便弄弄就搞定了，没费啥脑子（作者真棒！）。如果你也想用这个主题，可以直接到页面最下方找到 Astro & Pure theme 的链接。

过程中唯一比较费事的就是配置 waline 评论系统，遇到了经典的**跨域问题**，在此记录一番。

### 问题回顾

博客是静态的东西，所有的 markdown 文档都存储在代码仓库里，在新的 git push 之前是不会有任何内容变化的。而正常的前后端系统是有 db 的，每次操作都对 db 发送了写请求，所以才能做交互。

评论系统就是一个需要 db 的玩意，那我们懒得为了评论再去写一个后端，这时候就可以把人家写好的评论服务器（包括 db 操作和后端 api）部署下来用。本主题使用的评论系统是 [Waline](https://waline.js.org/)。主题作者将评论相关调用的 baseUrl 封装到了配置内，方便使用主题的用户部署好自己的 waline 后端后替换 url，太贴心了~

交互的路径大概是这样的：

![image-20250629071209909](https://typora-images-gqy.oss-cn-nanjing.aliyuncs.com/image-20250629071209909.png)

> 所以为啥不把 api 放到云上，直接做成通过 apikey 可得而是要让用户自己部署呢？

好，现在我已经在 vercel 上部署了 waline 后端系统，假设生成的默认域名是 **example.com**。那么此时我把该域名替换到 baseUrl 内，此时是无法正常显示数据的，因为出现了**跨域问题**。

![image-20250629072801797](https://typora-images-gqy.oss-cn-nanjing.aliyuncs.com/image-20250629072801797.png)

> 看到这经典的 CORS 报错，有没有感到很亲切~

### 关于跨域

这是浏览器的锅，一个 url 里面包括协议、域名、端口号，这三者全部一致才算同源。对于非同源的请求，就会被浏览器拦截。那么这个拦截对于简单请求与复杂请求其实要进一步区分。

> 复杂请求的条件包括使用 PUT、DELETE、PATCH 方法、自定义头、application/json Type 等等，比较杂。

对于简单请求：浏览器拿到了 server 的响应后将数据屏蔽。这种情况下，server 已经执行了请求。

![image-20250629085133741](https://typora-images-gqy.oss-cn-nanjing.aliyuncs.com/image-20250629085133741.png)

对于复杂请求：浏览器会先发送一个预检请求，如果发现不行了就不发正式请求了。这种情况下，server 没有执行请求。

![image-20250629085518572](https://typora-images-gqy.oss-cn-nanjing.aliyuncs.com/image-20250629085518572.png)

> 至于为什么会有跨域的屏蔽，这是出于安全性方面的考虑……但实际我们也看到，简单请求也可能更改服务端数据，而返回的响应只是被屏蔽了，服务端实际执行了请求。所以服务端数据安全还得在业务代码里自己保障。

### 问题解决

我在以前的大学课程项目中经常遇到这个问题，那时候前后端都是自己写的，那么在服务器的请求响应中增加 Access-Control-Allow-Origin 响应头即可。但是现在 waline 后端是写好的，我不想改，怎么办？

#### 子域名与同站

我先试了一个错误的方法：我的博客有一个域名 blog.com，我把 waline 后端增加了一个域名 waline.blog.com，这是 blog.com 的子域名。其实这显而易见的无法解决跨域问题，因为域名仍然不一样，blog.com 与 waline.blog.com 是非同源的。

可是 waline.blog.com 与 blog.com 总有些关系吧，那么这个关系的称呼叫作什么？

叫作同站。TLD（Top Level Domain，顶级域名）以及 TLD + 1 域名一致，就是同站 url。譬如 waline.blog.com 中，com 是 TLD，blog 是 TLD + 1，waline.blog.com 与 blog.com 就是同站请求。

同站请求虽然仍然会产生跨域拦截（因为不同源），但是与不同站的请求有一个区别，就是会自动携带 cookie。

> cookie 是什么玩意呢，其实就是一些 K-V，里面存储了一些登录状态与用户数据。假如你已经登录了 github.com，那么访问子域名 a.github.com，就会自动携带 cookie，从而直接进入登录状态。更详细的，请参考 [Cookie 的 SameSite 属性 - 阮一峰](https://www.ruanyifeng.com/blog/2019/09/cookie-samesite.html)。

#### 子路径解决跨域

看来我们必须保证 waline 的服务通过 https://blog.com/xxx 来访问了，这就是我们需要把 waline 服务映射到 blog.com 的子路径上。怎么做呢？我们必须有一个第三方为我们转发请求。

![image-20250629091622657](https://typora-images-gqy.oss-cn-nanjing.aliyuncs.com/image-20250629091622657.png)

如上图，浏览器并不知道发往 blog.com/waline 的请求被谁接手了，它只知道对方发回了想要的 response，同样的，waline 也不知道是谁发来的请求。第三方对于浏览器和 waline 都是透明的。

vercel 正好可以充当这个第三方，这种转发请求从而达到更改请求 url 的方法，被称为 rewrite。

将博客项目部署到 vercel 之后，我们只需要在博客项目内增加 vercel.json，如下：

```json
{
  "rewrites": [
    {
      "source": "/waline/:path*",
      "destination": "https://waline.blog.com/:path*"
    }
  ]
}
```

vercel 会自动解析，push 后重新部署即可。浏览器发出的请求被 vercel 转发，交给 waline 再传回，我们此刻就成功解决了跨域问题。

#### 其他

看到 rewrite，你可能会想到 redirect，这两个东西有什么区别呢？

![image-20250629092347462](https://typora-images-gqy.oss-cn-nanjing.aliyuncs.com/image-20250629092347462.png)

如上图，浏览器在 redirect 的情况下清楚地知道 server B 的存在，所以在跨域情景下，redirect 无法解决问题，只能是浏览器向 blog.com/waline 请求之后，再次向 waline.blog.com 请求，仍然会被无情拦截。

### 参考文章

1. [3 种CORS 错误的方式与 Access - Control - Allow - Origin 的作用原理](https://segmentfault.com/a/1190000022506474)
2. [同源与同站](https://juejin.cn/post/7233698667848777787)
3. [Cookie 的 SameSite 属性 - 阮一峰](https://www.ruanyifeng.com/blog/2019/09/cookie-samesite.html)
