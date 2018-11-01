---
title : 解决canvas在Retina屏上模糊问题
layout: post
date:   2017-10-17 
categories: 技术文章
---

##### 不废话直接上代码
```
* Retina-enable the given `canvas`.
 // 去除锯齿
      let canvas = document.querySelector('#gauge')
      const width = canvas.width,height=canvas.height;
      const cxt = canvas.getContext('2d')
      if (window.devicePixelRatio) {
        canvas.style.width = width + "px";
        canvas.style.height = height + "px";
        canvas.height = height * window.devicePixelRatio;
        canvas.width = width * window.devicePixelRatio;
        cxt.scale(window.devicePixelRatio, window.devicePixelRatio);
      };
```
