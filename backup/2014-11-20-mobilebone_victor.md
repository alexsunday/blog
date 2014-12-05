---
layout: post
title: 为Mobilebone摇旗
tags: [Javascript, Mobilebone]
keywords: Javascript, Mobilebone
---

{{ page.title }}
================ 

Mobilebone是一款为WebApp准备的简单的Js库，为WebApp而准备的Javascript已经有一打不止了，而且多数知名度较高，历史悠久如jQMobi，jQ UI，ExtJS，新晋之后如AngularJS，Ember.JS，轻量级如Backbone等，我们有必要再多弄个吗？

WebApp的痛点在哪里？

两个字即可概括：「体验」。

原始页面跳转、全刷新的技术对于体验自然是不必多说，新晋如AngularJS等的确扳回一城，但经过将Ember.JS嵌入Android Webview后，对一个压缩后仍有250KB左右的庞然大物心存畏惧，在鄙人的杂粮手机中，Ember.JS的加载长达七到八秒，着实「令人发指」。这还不提其对Js语言的各种「重定义」以及这带来的陡峭的学习曲线。老实说，有点复杂。

传统的Web开发，使用后端编程语言直接Render Template的方式，是Web开发最普遍使用的方式，大部分开发者比较熟悉此种方式。恰恰不够如意的是，这种方式只能做「网站」而不能做「WebApp」。

Mobilebone立志于此，希望简化WebApp的开发过程。极端情况下，使用原来的<a href/>的方式也可以开发出「单页面应用」。gzip压缩后的大小不过4KB以下而已。
