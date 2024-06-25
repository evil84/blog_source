# Light Sampling

在实现Path Tracing的过程中，除了根据BSDF做sampling之外，还会根据Light做sampling来实现MIS。这里简单记录下。

## Sampling a cone

采样基于球的面光源或者Spot Light时，我们希望可以在cone的方向均匀采样（uniform sampling)。其实类似半球采样，我们的$\phi$还是处于$0$到$2\pi$范围内，但是$\theta$不再处于$0$到$\frac{\pi}{2}$，而是处于$0$到$\theta_{max}$。简单推导一下pdf：

假设pdf为c，那么由pdf的定义可得：

$$
\int_{cone}cd\omega=1
$$
展开到球面积分：
$$
\int_{0}^{2\pi}\int_{0}^{\theta_{max}}csin(\theta)d\theta d\phi
$$
求解：
$$
\int_{0}^{2\pi}\int_{0}^{\theta_{max}}csin(\theta)d\theta d\phi
= \int_{0}^{2\pi}c(1-cos(\theta_{max}))d\phi
=2\pi c(1-cos(\theta_{max}))
= 1
$$
可知：
$$
pdf(\omega) = c = \frac{1}{2\pi(1-cos(\theta_{max}))}
$$
有0到1内的随机数$\epsilon$通过inverse method可知：
$$
cos(\theta) = (1 - \epsilon) + \epsilon cos(\theta_{max})
$$
写成代码（来自UE）：

```c++
float4 UniformSampleCone( float2 E, float CosThetaMax )
{
	float Phi = 2 * PI * E.x;
	float CosTheta = lerp( CosThetaMax, 1, E.y );
	float SinTheta = sqrt( 1 - CosTheta * CosTheta );

	float3 L;
	L.x = SinTheta * cos( Phi );
	L.y = SinTheta * sin( Phi );
	L.z = CosTheta;

	float PDF = 1.0 / ( 2 * PI * (1 - CosThetaMax) );

	return float4( L, PDF );
}
```

## Sampling sphere

采样点光源时，如果我们不知道着色点，这个时候对整个球进行uniform采样即可。这里不详述。这里主要记录一下在知道着色点的情况下，如何采样球形光源。

着色点$P$和光源中心$P_{c}$关系如下图：

![](E:\Projects\Blog\blog_source\source\pic\Light Sampling\1.png)

我们有任意$\theta \in \{0, \theta_{max}\}$，我们只需要计算出如下图所示夹角$\alpha$即可，这个时候以球的中心为原点，球中心到p点为z轴构建出本地坐标系，那么夹角$\alpha$即相当于上面cone的$\theta$，由推导sampling cone的方法，我们就能知道sampling sphere的方法。

![](E:\Projects\Blog\blog_source\source\pic\Light Sampling\2.png)

如下图所示，根据简单的三角函数计算，即可求出$\alpha$的值。

![](E:\Projects\Blog\blog_source\source\pic\Light Sampling\3.png)

再来看看UE，UE在Path Tracing的时候并不是以球为中心构建tangent坐标系，而是直接以着色点为中心，着色点到光源的向量为z轴构建Tangent坐标系。以着色点和光源采样点的方向和距离来构成sample点。所以，在UE里面，计算被简化为计算着色点和球上采样点的方向和距离。这时候我们只需要计算出上图的$sin(\theta_{max})$即可。看图可知，$sin(\theta_{max})$可以通过半径除以点到球心的距离算出来。

我们来看看UE怎么实现的：

```c++
float4 UniformSampleConeRobust(float2 E, float SinThetaMax2)
{
	float Phi = 2 * PI * E.x;
	float OneMinusCosThetaMax = SinThetaMax2 < 0.01 ? SinThetaMax2 * (0.5 + 0.125 * SinThetaMax2) : 1 - sqrt(1 - SinThetaMax2);

	float CosTheta = 1 - OneMinusCosThetaMax * E.y;
	float SinTheta = sqrt(1 - CosTheta * CosTheta);

	float3 L;
	L.x = SinTheta * cos(Phi);
	L.y = SinTheta * sin(Phi);
	L.z = CosTheta;
	float PDF = 1.0 / (2 * PI * OneMinusCosThetaMax);

	return float4(L, PDF);
}
```

这里的代码和上面的UniformSampleCone大同小异，主要是输入参数从$cos(\theta_{max})$变成了$sin^{2}(\theta_{nax})$。为什么函数后缀叫Robust呢。是因为计算$1-cos(\theta_{max})$的时候，$cos(\theta_{max})$是通过对$1-sin^{2}（{\theta_{max})}$开平方计算的。用$x$替代$sin^{2}（{\theta_{max})}$，可得$1-sqrt(1-x)$，当$x$即$sin^{2}（{\theta_{max})}$接近于0得时候容易导致Catastrophic cancellation（UE原注释）。这里UE采取用泰勒级数展开得方式将上式展开为多项式，避免了这个问题。这就是Robust的含义吧。我们这里用maxima验证一下，做对$1-sqrt(1-x)$关于x在0附近的2次展开：

![](E:\Projects\Blog\blog_source\source\pic\Light Sampling\4.png)

可以看出来和UE的代码保持一致。

