---
title: Sampling BRDF
date: 2021-06-16 22:00:00
categories: "Ray Tracing from the Ground Up"
tags: "Ray Tracing from the Ground Up"
mathjax: true
---

## 基本概念

1. $Pdf(\omega)$ : 立体角上的概率密度函数 （平常我们采样函数返回的pdf是这个）
2. $Pdf(\theta, \phi)$ : 球面坐标系上的概率密度函数 
3. $Pdf(\theta) = \int_0^\pi Pdf(\theta, \phi) d\phi$: $Pdf(\theta, \phi)$关于$\theta$的边缘概率密度函数
4. $Pdf(\phi | \theta) = \frac{Pdf(\theta, \phi)}{Pdf(\theta)}$: $\phi$关于$\theta$的条件概率密度函数
5. 我们有在(0, 1)范围内均匀分布概率的随机函数，生成的随机数我们定义为$\epsilon$,想计算出满足指定概率分布的随机数，这个时候需要用到逆变换采样(Inverse transform sampling)，首先对pdf求cdf：$cdf(x) = \int_{-\infty}^x{f(t)dt}$，然后令$\epsilon = cdf(x)$, 然后我们只需要求出$cdf^{-1}(x)$即是满足该概率分布的随机数。

## Uniform Sample Hemisphere

我们需要对整个半球均匀采样，假设我们不知道半球的概率密度函数是什么，我们设它为: 

$$Pdf(\omega) = c$$

,由概率定义可知，其在半球上的立体角的积分为1：

$$\int_{\Omega^2}{c}d\omega = 1$$

转换为球面积分：

$$\int_{\Omega^2}{c}sin(\theta){d\theta}{d\phi} = 1$$

求解：

$$\int_0^{\frac{\pi}{2}}{sin(\theta)} \int_0^{2\pi}d\phi d\theta = 2\pi c$$

要令$2\pi c = 1$,那么可知$c = \frac{1}{2\pi}$，可知，$Pdf(\omega) = \frac{1}{2\pi}$, $Pdf(\theta,\phi) = \frac{sin(\theta)}{2\pi}$

知道了$Pdf$,现在来计算采样向量，假设我们有2D随机变量E，由分层采样或者低偏差序列而来，x,y分别都在0到1之间随机分布，先计算关于$\theta$的边缘概率密度函数：

$$Pdf(\theta) = \int_0^{2\pi} \frac{1}{2\pi} sin(\theta) d\phi = sin(\theta)$$

然后计算累积分布函数：

$$CDf(\theta) = \int_0^{\theta} sin(t) dt = 1 - cos(\theta)$$

令$\epsilon_1 = 1 - cos(\theta)$，可知$cos(\theta) = 1 - \epsilon_1$，又因$\epsilon_1, \epsilon_2$都在$(0..1)$范围内均匀分布，故可以用$\epsilon_1$替换$1-\epsilon_1$,这样可知$\theta = cos^{-1}(\epsilon_1)$。

继续求$Cdf(\phi | \theta)$：

$$Pdf(\phi | \theta) = \frac{Pdf(\theta, \phi)}{Pdf(\theta)} = \frac{\frac{sin(\theta)}{2\pi}}{sin(\theta)} = \frac{1}{2\pi}$$

$$Cdf(\phi | \theta) = \int_0^{\phi} \frac{1}{2\pi} dt = \frac{\phi}{2\pi}$$

令$\epsilon_2 = \frac{\phi}{2\pi}$, 可知$\phi = 2 \pi \epsilon_2$。

转换到球面坐标系，我们这里用UE的坐标系，z轴向上，球的半径为1，那么笛卡尔坐标系和球面坐标系的关系为：

$$ x = sin(\theta) cos(\phi) $$

$$ y = sin(\theta) sin(\phi) $$

$$z = cos(\theta)$$

令$E.x = \epsilon_2, E.y = \epsilon_1$, 可知：

$cos(\theta) = E.y$

$sin(\theta) = \sqrt[2]{1 - E.y * E.y}$

$$cos(\phi) = cos(2 *\pi * E.x)$$

$$sin(\phi) = sin(2 *\pi * E.x)$$

这样就可以求出$x,y,z$分别的值了。

UE5代码参考：

```c++
float4 UniformSampleHemisphere( float2 E )
{
	float Phi = 2 * PI * E.x;
	float CosTheta = E.y;
	float SinTheta = sqrt( 1 - CosTheta * CosTheta );

	float3 H;
	H.x = SinTheta * cos( Phi );
	H.y = SinTheta * sin( Phi );
	H.z = CosTheta;

	float PDF = 1.0 / (2 * PI);

	return float4( H, PDF );
}
```



## Cosine Weight Sample Hemisphere

通常我们采样时，会根据brdf的形状生成想要的概率分布，常用的漫反射模型如lambert，oren-nayar等都可以用这个分布去采样，他们的特点是平行于法线的贡献高，所以需要在这附近多生成射线，越靠近垂直于法线的贡献低，生成的射线越少，也算是重要性采样(importance sampling)的一种。

同样，我们假设不知道$pdf$是什么，考虑到$cos(\theta)$的加权，我们假设为：

$$Pdf(\omega) = c cos(\theta) $$

立体角上的半球积分，：

$$\int_{\Omega^2} c cos(\theta) d{\omega} = 1$$

转换到球面积分：

$$\int_{\Omega^2}c cos(\theta) sin(\theta){d\theta}{d\phi} = 1$$

求解：

$$\int_0^{\frac{\pi}{2}}{c cos(\theta) sin(\theta)} \int_0^{2\pi}d\phi d\theta = \pi c$$

要令$\pi c = 1$，那么可知$c = \frac{1}{\pi}$, 考虑到$cos(\theta)$加权，可知$Pdf(\omega) = \frac{cos(\theta)}{\pi}$，  $Pdf(\theta, \phi) = \frac{\cos(\theta) \sin(\theta)}{\pi}$。

仿照上例，先计算关于$\theta$的边缘概率密度函数：

$$Pdf(\theta) = \int_0^{2\pi} \frac{cos(\theta)}{\pi} sin(\theta) d\phi = 2 cos(\theta)sin(\theta)$$

然后计算累计分布函数：

$$Cdf(\theta) = \int_0^{\theta}2cos(\theta)sin(\theta) = 1-{{\cos^2{\left( \theta \right) }}}$$

令$\epsilon_1 = 1 - cos^2(\theta)$，可知$\theta = cos^{-1}\sqrt{1 - \epsilon_1}$，同理用$\epsilon_1$替换$1-\epsilon_1$，可得$\theta = cos^{-1}\sqrt{\epsilon_1}$

继续求$Cdf(\phi | \theta)$：

$$Pdf(\phi | \theta) = \frac{Pdf(\theta, \phi)}{Pdf(\theta)} = \frac{\frac{\cos(\theta) \sin(\theta)}{\pi}}{2 cos(\theta)sin(\theta)} = \frac{1}{2\pi}$$

$$Cdf(\phi | \theta) = \int_0^{\phi} \frac{1}{2\pi} dt = \frac{\phi}{2\pi}$$

令$\epsilon_2 = \frac{\phi}{2\pi}$, 可知$\phi = 2 \pi \epsilon_2$。

知道$\theta$和$\phi$的值，其他计算同上，这里就省略了。

UE5代码参考：
```c++
float4 CosineSampleHemisphere( float2 E )
{
	float Phi = 2 * PI * E.x;
	float CosTheta = sqrt(E.y);
	float SinTheta = sqrt(1 - CosTheta * CosTheta);

	float3 H;
	H.x = SinTheta * cos(Phi);
	H.y = SinTheta * sin(Phi);
	H.z = CosTheta;

	float PDF = CosTheta * (1.0 / PI);

	return float4(H, PDF);
}
```


## Microfacet BRDF Sampling

接下来考虑微表面的情况，按Cook-Torrance公式来说，我们知道，对Specular Lobe形状影响最大的主要来自于法线分布函数（Normal Distribute Function)，所以我们需要生成满足法线分布函数分布概率的采样向量，注意，NDF并不完全等于概率密度函数，需要考虑到微观表面到宏观表面的投影，这里我们就不展开了，直接列公式（这里$D(m)$代表法线分布函数NDF）：

$$\int_{\Omega^2}D(m)cos(\theta_m)d\omega = 1$$

下面我们把常见的NDF都推导一遍，同时需要考虑各向同性和各项异性两种情况。

### GGX isotropic

$$D_{GGX}(m,\alpha) = \frac{a^2}{\pi ( cos^2(\theta)(a^2 - 1) + 1)^2} $$

$$Pdf(\omega) = \frac{a^2 cos(\theta)}{\pi ( cos^2(\theta)(a^2 - 1) + 1)^2}$$

$$Pdf(\theta, \phi) = \frac{a^2 cos(\theta) sin(\theta)}{\pi ( cos^2(\theta)(a^2 - 1) + 1)^2}$$

$$Pdf(\theta) = \int_0^{2\pi}\frac{a^2 cos(\theta) sin(\theta)}{\pi ( cos^2(\theta)(a^2 - 1) + 1)^2}d\phi = \frac{2 {{a}^{2}} \cos{(\theta)} \sin{(\theta)}}{{{\left( \\cos^2{(\theta)} ({{a}^{2}}-1)+1\right) }^{2}}}$$ 

$$Pdf(\phi|\theta)= \frac{Pdf(\theta,\phi)}{Pdf(\theta)} = \frac{\frac{a^2 cos(\theta) sin(\theta)}{\pi ( cos^2(\theta)(a^2 - 1) + 1)^2}}{\frac{2 {{a}^{2}} \cos{(\theta)} \sin{(\theta)}}{{{\left( \cos^2{(\theta)} ({{a}^{2}}-1)+1\right) }^{2}}}} = \frac{1}{2\pi}$$

$$Cdf(\theta) = \int_0^{\theta}\frac{2 {{a}^{2}} \cos{(t)} \sin{(t)}}{{{\left( \\cos^2{(t)} ({{a}^{2}}-1)+1\right) }^{2}}}dt = -\frac{cos^2(\theta) - 1}{( {{a}^{2}}-1)  {{cos^2{( \theta ) }}}+1}$$

令$\epsilon = -\frac{cos^2(\theta) - 1}{( {{a}^{2}}-1)  {{cos^2{( \theta ) }}}+1}$，可知$\theta = cos^{-1}\left( \sqrt{\frac{1-\epsilon}{\left( {{a}^{2}}-1\right)  \epsilon+1}}\right)$

求解$\phi$同上，这里就不赘述了。

UE5代码参考：

```c++
float4 ImportanceSampleGGX( float2 E, float a2 )
{
	float Phi = 2 * PI * E.x;
	float CosTheta = sqrt( (1 - E.y) / ( 1 + (a2 - 1) * E.y ) );
	float SinTheta = sqrt( 1 - CosTheta * CosTheta );

	float3 H;
	H.x = SinTheta * cos( Phi );
	H.y = SinTheta * sin( Phi );
	H.z = CosTheta;
	
	float d = ( CosTheta * a2 - CosTheta ) * CosTheta + 1;
	float D = a2 / ( PI*d*d );
	float PDF = D * CosTheta;

	return float4( H, PDF );
}
```
### GGX anisotropic

$$D_{GGX\_aniso}(m,\alpha) = \frac{1}{\pi \alpha_t \alpha_b} \frac{1}{((\frac{t \cdot m}{\alpha_t})^2 + (\frac{b \cdot m}{\alpha_b})^2 + (n \cdot m)^2)^2}$$


### beckmann isotropic

### beckmann anisotropic

### blinn phong









