---
title: 我为什么选择将博客托管到 GitHub Pages 上
date: 2018-08-17 19:39:11
lastmod: 2019-10-30
tags: [运维, 博客]
---

从大一开始写博客以来，我使用过 [CSDN](https://blog.csdn.net/MeiK_SDUT)、[Typecho](http://typecho.org/) 以及现在的 [hexo](https://hexo.io/) + [GitHub Pages](https://pages.github.com/)。我不是一个喜欢频繁变动的人，每次更换博客系统都是因为上一个博客有让我无法忍受的问题。
<!--more-->

## CSDN 时期

从大一寒假开始到大三寒假。我从大一开始接触编程，平时喜欢自己写一些小玩具什么的，一直是发到 QQ 群里。大一寒假我在过年前几天用 C 语言写了一个[贪吃蛇游戏](https://blog.csdn.net/MeiK_SDUT/article/details/50650098)，发到群里，好评如潮。有同学问我是怎么实现的，我就顺势注册了 CSDN 的账号，将代码贴了上去。

之后开学，参加了校内的 ACM 集训队，会偶尔把自己做的题的题解贴到博客上。我平时做题很少写题解博客，只有哪天想起来了，才会随便找一个题写个题解...因此这之后的半年多，我基本保持着每个月一篇博客的速度。

退队之后去了运维中心做开发，持续了有两年之久，这期间基本没有更新博客...等到快要到春招的时候，我又准备拾起来我的博客，这时候 CSDN 的种种问题都浮现出来：比如铺天盖地的广告、稀烂的排版、对 markdown 的巨弱支持等等。让我下定决心跑路的是 CSDN 神一般的交互，作为一个写博客的网站，我需要绕好几个圈才能到写博客的页面（这个问题现在有所好转，但是如果想要看自己关注的人的话，还是要绕几个让人摸不着头脑的圈子）。然后我就决定自建一个博客。

## Typecho 时期

大三寒假期间，我因为无法忍受 CSDN ，选择自建博客。之所以选择了 Typecho ，是因为有另一个同学使用了这个系统，然后我网上看了它的评价也很不错。

Typecho 是使用 PHP 开发的，部署和扩展都很方便。我感到很满意，就这么用了半年左右，然后就发现了几个问题。

Typecho 原生对 markdown 的支持并不好，尤其是目录和页内锚点，并不能正常的渲染出来。

用了好几天在网上找解决方案、自己写插件，最后用了一套麻烦的解决方案，用了好几个插件。虽然表面上来看问题解决了，但是没解决的问题还是有很多。

一开始我换了一套 markdown 的解析器，然后发现这套解析器虽然能解析页内锚点了，但是它的其他方面的问题更严重，严重到无法使用的地步。比如不能使用目录，原生的解析器虽然不能解析目录，但页面大体结构是可以渲染出来的。这套解析器就厉害了，只要使用目录，整个页面都会瞬间爆炸。还有就是它不能解析多层的 `ul` 和 `li` ，导致整个页面全乱了。（而且还是 star 很多的一个库）

后来我使用了一套基于 marked 的解析器，这个果然好用，之前的所有问题都瞬间消失了。然而，如果没有任何问题的话，我就不会沦落到去更换系统的地步了...

这套系统解析是没有问题的，但是！它需要整个页面全部加载完成之后才会显示博客正文！也就是说，在页面完全加载完成之前，整个正文段全是空白的！我的服务器的小水管，导致加载比较缓慢，进而就导致会显示很久的空白，体验极差。

## GitHub Pages + Hexo

最后我换了这套，虽然不支持动态网页，不能原生实现评论等功能，但是页面足够干净，功能也足够我使用。还有一点很重要，GitHub 与 Let's Encrypt 合作，所有的 GitHub Pages 都会自动支持 https ！

虽然不知道我以后还会不会再次更换我的博客系统，但就目前来说，这套系统绝对是我最喜欢的一套了。干净的布局，没有让人摸不着头脑的奇怪逻辑，估计我会使用这套系统相当久吧。

## GitHub Pages + Hugo + Cloudflare

时隔一年半，我将我的博客系统由 Hexo 切换到了 Hugo，因为 Hugo 更快，在博客与静态资源达到一定数量之后，Hexo 开始明显的变慢了。而切换到 Hugo 之后，每次生成页面的时间由原来的几十秒减少到了毫秒级别，配合上我自己写的博客样式，再战几年没有问题。
