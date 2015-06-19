---
layout: post
title:  ray tracing (Unfinished)
date:   2015-06-19 10:00
categories: coding
---

## 1. Introduction
光线追踪是一个全局光照渲染方法。简单地说，就是从摄像机的位置，通过影像平面上的像素位置（比较正确的说法是取样（sampling）位置），发射一束光线到场景，求光线和几何图形间最近的交点，再求该交点的着色。如果该交点的材质是反射性的，可以在该交点向反射方向继续追踪。光线追踪除了容易支持一些全局光照效果外，也不局限于三角形作为几何图形的单位。任何几何图形，能与一束光线计算交点（intersection point）,就能支持。

![ray_trace](/images/ray_trace.png)

上图[来源](https://en.wikipedia.org/wiki/File:Ray_trace_diagram.svg)