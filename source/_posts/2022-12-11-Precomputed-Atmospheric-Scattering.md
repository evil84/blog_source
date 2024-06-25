---
title: Precomputed Atmospheric Scattering
date: 2022-12-11 02:24:00
categories: "Realtime Rendering"
tags: "atmosphere scattering"
mathjax: true
---

这篇文章主要是对[这段源码](https://ebruneton.github.io/precomputed_atmospheric_scattering/atmosphere/functions.glsl.html)的解读，个人觉得理解了这段代码，就对Precomputed Atmospheric Scattering算法有了比较深入的了解，对于不清楚其中一些术语的读者可以先看看[这个系列文章](https://www.alanzucconi.com/2017/10/10/atmospheric-scattering-1/)

## Transmittance

### Computation

先看看这幅图，我们要计算点**q**到**p**的Transmittance，我们知道**i**到**p**的Transmittance等于**i**到**q**的Transmittance乘以**q**到**p**的Transmittance，所以**q**到**p**的Transmittance等于**i**到**p**的Transmittance除以**i**到**q**的Transmittance。从而可知，想计算任意两点之间的Transmittance，只需要计算每个点到大气层边界的Transmittance，然后相除就好了。

![](E:\Projects\Blog\blog_source\source\pic\2022-12-11-Precomputed-Atmospheric-Scattering\Snipaste_2022-12-13_20-15-05.png)

要计算大气层内任意一点**p**沿着向量$\vec{pq}$到大气层边界的Transmittance，由图可以观察我门只需要知道点**p**离地心**o**的距离**r**和向量$\vec{op}$和$\vec{pq}$的夹角的点积**u** = cos($\theta$)就可以计算出来了。设**p**到**i**的距离为**d**，由三角数学可知，**i**点的坐标为：

**x**：dsin($\theta$) = d$\sqrt{(1-cos^2(\theta))}$ = d$\sqrt{1-u^2}$
**z**：r + dcos($\theta$) = r + du

大气层的半径为：$r_{top}$。

我们先计算与大气层的距离：

$x^2 + z^2 = r^2_{top}$

$d^2 (1-u^2) + (r + du)^2 = r^2_{top}$

$d^2 - d^2u^2 + r^2 + 2rdu + d^2u^2 = r^2_{top}$

$d^2 + 2rdu + r^2 - r^2_{top} = 0$

这是个以d为变量的二元一次方程，解方程：

$d = \frac{-2ru + \sqrt{4r^2u^2 - 4(r^2 - r^2_{top})} }{2} = -ru + \sqrt{r^2 (u^2-1) + r^2_{top}} $ 

``` c++
Length DistanceToTopAtmosphereBoundary(IN(AtmosphereParameters) atmosphere,
    Length r, Number mu) {
  assert(r <= atmosphere.top_radius);
  assert(mu >= -1.0 && mu <= 1.0);
  Area discriminant = r * r * (mu * mu - 1.0) +
      atmosphere.top_radius * atmosphere.top_radius;
  return ClampDistance(-r * mu + SafeSqrt(discriminant));
}
```

同理，我们可以计算与星球的距离：

``` c++
Length DistanceToBottomAtmosphereBoundary(IN(AtmosphereParameters) atmosphere,
    Length r, Number mu) {
  assert(r >= atmosphere.bottom_radius);
  assert(mu >= -1.0 && mu <= 1.0);
  Area discriminant = r * r * (mu * mu - 1.0) +
      atmosphere.bottom_radius * atmosphere.bottom_radius;
  return ClampDistance(-r * mu - SafeSqrt(discriminant));
}
```









