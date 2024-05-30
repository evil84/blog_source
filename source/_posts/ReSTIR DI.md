# ReSTIR DI

## Importance Sampling

## Multiple Importance Sampling (MIS)

## Resampled importance sampling (RIS)

理论上我们选取的样本与被积函数成比例，这样可以得到最好的结果。但实际上，有时候这很困难。在渲染公式里，这通常与V（可见项）相关（假如我们没有实际上发射光线，我们怎么知道是否可见呢）。然而，我们可以抽取与被积函数的部分项成比例的样本，比如我们忽略掉V项，只考虑BSDF和Incoming Radiance。我们采用以下方法生成样本。

1、我们从一个次优但是易于计算的分布里（比如半球分布）生成M>=1个候选样本，$x={\{x_1,...,x_M}\}$，然后再从这M个样本中根据索引$z∈\{{1,...M}\}$随机抽取一些样本，$\hat{p}$为目标函数（比如Radiance*BSDF，可以理解为被积函数的简化版，方便计算，比如不需要在为每个着色点对光源做Ray Tracing求交），每个样本的独立概率为：
$$
\begin{align*}
p(z|x)&=\frac{w(x_z)}{∑^{M}_{i=1}w(x_i)} & w(x) &= \frac{\hat{p}(x)}{p(x)}
\end{align*}
$$

2、我们有被积函数$f$，目标函数$\hat{p}$，$M$个样本，每个样本的权重$w$，样本$y∈x_z$，1-sample的RIS估值器（estimator）为：
$$
\langle L \rangle^{1,M}_{ris}=\frac{f(y)}{\hat{p}(y)}\cdot\Bigg(\frac{1}{M}\sum^M_{j=1}w(x_j)\Bigg)
$$
3、N-sample的RIS估值器就是简单的对N个1-sample的RIS估值器的结果求平均：
$$
\langle L \rangle^{N,M}_{ris}=\frac{1}{N}\sum^N_{i=1}\bigg(\frac{f(y_i)}{\hat{p}(y_i)}\cdot\Bigg(\frac{1}{M}\sum^M_{j=1}w(x_{ij})\Bigg)\Bigg)
$$

## Weighted Reservoir Sampling

听起来很复杂，实际很简单，假设我们有N个样本${x_1, x_2,...x_m,...x_n}$，每个样本对应一个权重$w(x_i)$，那么，我们选中每个样本的概率为：
$$
p_i=\frac{w(x_i)}{\sum^M_{j=1}w(x_j)}
$$

既是说，我们选中每个样本的概率为当前样本的权重除以当前样本之前（包含当前样本）所有权重的和。

## Streaming RIS with spatio temporal Reuse

通过RIS和WRS的结合，我们可以复用时间和空间上的样本（邻近像素和历史帧的像素），从而不需要太多额外的运算就可以提供数量级以上的有效样本数，类似SpatioTemporal滤波，但是朴素的复用时空上的样本会导致结果不再是无偏的。因为不同的像素选取样本基于不同的BRDF和面的朝向，几何上的不连续会导致能量损失。后面会讲到方法来避免这种问题。







