[TOC]

非线性回归
---

**支持向量机(Support Vector Machine)**简称**SVM**在学习复杂的非线性方程时提供了一种更为清晰，更加强大的方式。

### 支持向量机引入

在逻辑回归中，假设函数为：
$$
h_\theta(x) = \frac{1}{1 + e^{-\theta^T x}}
$$
对应的$J(\theta)$中的$cost$为：
$$
-(ylog(h_\theta(x)) + (1-y)log(1-h_\theta(x))) = -ylog( \frac{1}{1 + e^{-\theta^T x}}) - (1-y)log( \frac{1}{1 + e^{-\theta^T x}})
$$
在$y = 1$和$y=0$的两种情况下，$cost$函数关于$z$的图：

![逻辑回归cost关于z的函数](http://gtw.oss-cn-shanghai.aliyuncs.com/machine-learning/Stanford/%E9%80%BB%E8%BE%91%E5%9B%9E%E5%BD%92cost%E5%85%B3%E4%BA%8Ez%E5%87%BD%E6%95%B0.jpeg)

现在开始支持向量机，画一个非常接近逻辑回顾函数的折线，这个折线由$z=1$(或$z=-1$)一点的两条线段组成：

![逻辑回归cost的折线](http://gtw.oss-cn-shanghai.aliyuncs.com/machine-learning/Stanford/%E9%80%BB%E8%BE%91%E5%9B%9E%E5%BD%92cost%E7%9A%84%E6%8A%98%E7%BA%BF.jpeg)

现在命名上图对应的两个方程，$y=1$左图为$cost_1(z)$，$y=0$右图为$cost_0(z)$。

### 构建支持向量机

1. 替换逻辑回归中的代价函数$J(\theta)$，$cost_1(z)$和$cost_0(z)$就是上面提到的那两条靠近逻辑回归函数的折线，得到支持向量机的最小化代价函数：
   $$
   J_{min}(\theta) = \frac{1}{m}\sum_{i=1}^{m}[y^{(i)}cost_1(\theta^Tx^{(i)}) + (1-y^{(i)})cost_0((\theta^Tx^{(i)})] + \frac{\lambda}{2m}\sum_{j=1}^{n}\theta_j^2
   $$

2. 按照**支持向量机**的惯例，去除$\frac{1}{m}$这一项，因为这一项是个常数项，即使去掉也可以得出相同的$\theta$最优值：
   $$
   min_\theta \sum_{i=1}^{m}[y^{(i)}cost_1(\theta^Tx^{(i)}) + (1-y^{(i)})cost_0((\theta^Tx^{(i)})] + \frac{\lambda}{2}\sum_{j=1}^{n}\theta_j^2
   $$

3. 正则化项系数处理

   对于线性回归和逻辑回归，通过修改不同的正则化参数$\lambda$来达到优化目的，这样就能够使得训练样本拟合的更好。但对于支持向量机，按照惯例将使用一个不同的参数来替换这里使用的$\lambda$来实现权衡这两项的目的。这个参数称为**C**。同时将优化目标改为$CA + B$形式:

$$
min_\theta\ C\sum_{i=1}^{m}[y^{(i)}cost_1(\theta^Tx^{(i)}) + (1-y^{(i)})cost_0((\theta^Tx^{(i)})] + \frac{1}{2}\sum_{j=1}^{n}\theta_j^2
$$

在逻辑回归中，如果给$\lambda$一个很大的值，那么就意味着给与$B$了一个很大的权重，而在在**支持向量机**中，就相当于对$C$设定了一个非常小的值(<mark>C相当于$\frac{1}{\lambda}$</mark>)，这样一来就相当于对$B$给了比$A$更大的权重。因此，这只是一种来控制这种权衡关系的不同的方式。

最后有别于**逻辑回归**的一点，逻辑回归$h_\theta(x) =  \frac{1}{1+e^{-\theta^T x}}$，而对于**支持向量机**假设函数的形式如下：
$$
\begin{eqnarray*}
h_\theta(x) = 1 & & if \ \theta^Tx \geq 0 \\
h_\theta(x) = 0 & & if \ \theta^Tx <0
\end{eqnarray*}
$$

### 大间距分类器

关于支持向量机模型的代价函数

- 如果你有一个正样本，即$y=1$时，那么代价函数$cost_1(z)$的图像如下:

  ![支持向量机cost1](http://gtw.oss-cn-shanghai.aliyuncs.com/machine-learning/Stanford/%E6%94%AF%E6%8C%81%E5%90%91%E9%87%8F%E6%9C%BAcost1.png)

  **只有在$z \geq 1$(即$\theta^Tx \geq 1$)时，代价函数$cost_1(z)$的值才等于0**。

- 反之，如果你有一个负样本，即$y=0$时，那么代价函数$cost_0(z)$的图像如下：

  ![支持向量机cost0](http://gtw.oss-cn-shanghai.aliyuncs.com/machine-learning/Stanford/%E6%94%AF%E6%8C%81%E5%90%91%E9%87%8F%E6%9C%BAcost0.png)

  **只有在$z \leq −1$(即$\theta^Tx \leq -1$)时，代价函数$cost_0(z)$的值才等于0**。

在逻辑回归中，仅需要$\theta^Tx \geq 0$或$\theta^Tx < 0$，但是**支持向量机**的要求更高，这里要求$\theta^Tx \geq 1$以及$\theta^Tx \leq -1$。相当于在**支持向量机**中嵌入了一个额外的安全因子（或者说是安全距离因子）。

#### 安全距离因子

具体而言，将代价函数中的常数项$C$设置成一个非常大的值，则最小化代价函数的时候，很希望找到一个使第一项$\sum_{i=1}^{m}[y^{(i)}cost_1(\theta^Tx^{(i)}) + (1-y^{(i)})cost_0((\theta^Tx^{(i)})]$为0的最优解。

当输入一个正样本$y^{(i)}=1$时，对于代价函数$cost_1(z)=0$需要使得当$\theta^Tx^{(i)} \geq 1$；对于一个负训练样本$y^{(i)}=0$时，对于代价函数$cost_0(z)=0$需要使得$\theta^Tx^{(i)} \leq -1$。最终只有$min\frac{1}{2}\sum_{j=1}^{n}\theta_j^2$，这样就会得到了一个非常有趣的决策边界。

#### SVM决策边界

下图中黑色直线看起来好很多，因为它看起来更加稳健。在数学上来讲就是这条直线拥有相对于训练数据更大的最短距离，这个所谓的距离就是指**间距(margin)**：

![SVM决策边界](http://gtw.oss-cn-shanghai.aliyuncs.com/machine-learning/Stanford/SVM%E5%86%B3%E7%AD%96%E8%BE%B9%E7%95%8C.png)

而粉线和蓝线距离训练样本非常近，在分离样本时就会表现的比黑线差。这就是**支持向量机**拥有[鲁棒性](http://baike.baidu.com/link?url=My7Y1mL_9uj-XdR2DC2kyGLop-AaPdzSgdNmgRmaJVYV77puxNs-_A7ERwLv3uWih02JCu6esljRn90mc3EkMwKXBckm_6wSqU42EX06vC4ouxQhinIZco7crxr7HetC)的原因。因为它一直<mark>努力用一个最大间距来分离样本。因此支持向量机分类器有时又被称为**大间距分类器**</mark>。

当有一个异常值产生时，因为正则化因子$C$设置的非常大，决策边界会产生偏移。但如果适当的减小$C$的值，最终还是会得到那条黑色的决策边界的。<mark>如果数据是线性不可分的话，支持向量机也可以恰当的将它们分开</mark>。

### 大间距分类器数学原理

#### 向量内积

$u^Tv$的计算结果称为$u$和$v$的内积，理解为：$v$向量在$u$向量上的投影长度$p$($p$是有方向的)乘以$u$的长度：
$$
u^Tv = p \cdot ||u||=v^Tu
$$

#### SVM决策边界

在支持向量机中的目标函数:
$$
min\frac{1}{2}\sum_{j=1}^{n}\theta_j^2 = \frac{1}{2}||\theta||^2
$$
可见，**<mark>支持向量机所做的事情，其实就是在极小化参数向量$\theta$范数的平方</mark>（或者说是长度的平方）**。
$$
\theta^Tx^{(i)} = p^{(i)} \cdot ||\theta||
$$
对于满足$X\theta = 0$表达式的$X$这些点连成的线就叫决策边界，可以看出<mark>$X$与$\theta$是相互垂直的</mark>。假设有下图绿色的这样一条决策边界，可以绘制出对应的参数向量θ(假设$\theta_0=0$)：

![SVM决策边界1](http://gtw.oss-cn-shanghai.aliyuncs.com/machine-learning/Stanford/SVM%E5%86%B3%E7%AD%96%E8%BE%B9%E7%95%8C1.jpeg)

可以发现这些$p^{(i)} $投影在$\theta$上是一些非常小的数，需要$p^{(i)} \cdot ||\theta|| \geq 1$或$p^{(i)} \cdot ||\theta|| \leq -1$，则意味着$||\theta|| $需要非常大，而现实目标是希望找到一个参数$\theta$使$||\theta|| $尽可能小。因此上图展示的并不是一个好的决策边界。

![SVM决策边界2](http://gtw.oss-cn-shanghai.aliyuncs.com/machine-learning/Stanford/SVM%E5%86%B3%E7%AD%96%E8%BE%B9%E7%95%8C2.jpeg)

如果支持向量机选择了上图这个决策界,状况会有很大不同。这个绿色的决策界有一个垂直于它的向量$\theta$，做同样的投影可以发现现在$p^{(i)} $在$\theta$上这些投影长度长多了。

要想$\theta$的范数$||\theta|| $最小，就得投影$p$的长度变大。这就是**为什么支持向量机可以产生大间距分类的原因**。

![SVM决策边界3](http://gtw.oss-cn-shanghai.aliyuncs.com/machine-learning/Stanford/SVM%E5%86%B3%E7%AD%96%E8%BE%B9%E7%95%8C3.jpeg)

即便$\theta_0 \neq 0$，支持向量机仍然会找到正样本和负样本之间的大间距分隔。这就解释了为什么支持向量机是一个大间距分类器。

### 核函数(Kernels)

之前做非线性决策边界是由类似下面这种多项式构成：
$$
\theta_0+\theta_1x_1+\theta_2x_2+\theta_3x_1x_2+\theta_4x_1^2+\theta_5x_2^2+\cdots=0
$$
可以把假设函数改写成下面这种形式：
$$
\theta_0+\theta_1f_1+\theta_2f_2+\theta_3f_3+\theta_4f_4+\theta_5f_5+\cdots=0
$$
以此类推可以依次加入这些高阶项，但我们其实并不知道这些高阶项是不是我们真正需要的。如果基础特征非常多，得到高阶项后续运算量将非常大。

#### 核函数构造新特征

![Kernel](http://gtw.oss-cn-shanghai.aliyuncs.com/machine-learning/Stanford/kernel.png)

有一个可以构造新特征$f_1$、$f_2$、$f_3$的方法：
$$
f_1 = similarity(x,l^{(1)})=exp(-\frac{||x-l^{(1)}||^2}{2\sigma^2})=exp(-\frac{\sum_{j=1}^{n}(x_j-l_j^{(l)})^2}{2\sigma^2})
$$

```matlab
function sim = gaussianKernel(x1, x2, sigma)

% Ensure that x1 and x2 are column vectors
x1 = x1(:); x2 = x2(:);

% return the following variables
sim = 0;


p = (x1 - x2);
sim = exp(-1 * ((p' * p) / (2 * (sigma^2))));
    
end
```

这里的$similarity(x,l)$函数，就被称为**核函数(Kernels)**，<mark>通常会被写作$k(x,l{(i)})$</mark>。在这里例子中所说的**核函数**，实际上是**高斯核函数**，在后面还会见到不同的**核函数**。

#### 深入理解核函数

$l^{(1)}$是之前在图中选取的几个点之中的一个，上述公式上面是$x$和$l^{(1)}$之间的核函数。其中$||x-l^{(1)}||^2$这一项可以表示成各个$x$向量到$l^{(1)}$向量的距离求和:$\sum_{j=1}^{n}(x_j-l_j^{(l)})^2$。

##### $x$对$f$值的影响

如果$x \approx l^{(1)}$，即$x$与其中一个标记点非常接近，那么这个欧氏距离$||x-l^{(1)}||^2$就会接近0，则$f_1 \approx exp(-\frac{0^2}{2\sigma^2}) \approx 1$。

如果$x$距离$l^{(1)}$很远，则$f_1 \approx exp(-\frac{(large \ number)^2}{2\sigma^2}) \approx 0$。

给出一个训练样本$x$，**基于之前给出的标记点$l^{(1)}$、$l^{(2)}$、$l^{(3)}$来计算出三个新的特征变量$f_1$、$f_2$、$f_3$。**

<mark>**这些特征变量$f$的作用是度量$x$到标记$l^{(1)}$的相似度的，并且如果$x$离$l$非常接近，那么特征变量$f$就接近1；如果$x$离标记$l^{(1)}$非常远，那么特征变量$f$就接近于0。**</mark>

##### $\sigma^2$对$f$值的影响

|             $\sigma^2 = 0.5$             |              $\sigma^2 = 1$              |              $\sigma^2 = 3$              |
| :--------------------------------------: | :--------------------------------------: | :--------------------------------------: |
| ![高斯核函数中sigma对f的影响1](http://gtw.oss-cn-shanghai.aliyuncs.com/machine-learning/Stanford/%E9%AB%98%E6%96%AF%E6%A0%B8%E5%87%BD%E6%95%B0%E4%B8%ADsigma%E5%AF%B9f%E7%9A%84%E5%BD%B1%E5%93%8D1.png) | ![高斯核函数中sigma对f的影响2](http://gtw.oss-cn-shanghai.aliyuncs.com/machine-learning/Stanford/%E9%AB%98%E6%96%AF%E6%A0%B8%E5%87%BD%E6%95%B0%E4%B8%ADsigma%E5%AF%B9f%E7%9A%84%E5%BD%B1%E5%93%8D2.png) | ![高斯核函数中sigma对f的影响3](http://gtw.oss-cn-shanghai.aliyuncs.com/machine-learning/Stanford/%E9%AB%98%E6%96%AF%E6%A0%B8%E5%87%BD%E6%95%B0%E4%B8%ADsigma%E5%AF%B9f%E7%9A%84%E5%BD%B1%E5%93%8D3.png) |

通过3D图可以看出，$\sigma^2$越小，凸起宽度越窄；而$\sigma^2$越大，则凸起宽度也扩大了。

#### 获取预测函数

![高斯核函数预测步骤1](http://gtw.oss-cn-shanghai.aliyuncs.com/machine-learning/Stanford/%E9%AB%98%E6%96%AF%E6%A0%B8%E5%87%BD%E6%95%B0%E9%A2%84%E6%B5%8B%E6%AD%A5%E9%AA%A41.png)

假设已经得到了这些参数$\theta$的值，现在有一个训练样本$x$如上图：训练样本$x$接近于$l^{(1)}$，那么$f_1$就接近于1;训练样本$x$离$l^{(2)}$、$l^{(3)}$都很远，所以$f_2$就接近于0，$f_3$也接近于0。带入公式得：
$$
\theta_0+\theta_1f_1+\theta_2f_2+\theta_3f_3 = -0.5 + 1 = 0.5 \geq 0
$$
所以对这一点预测结果为$y=1$。

![高斯核函数预测步骤2](http://gtw.oss-cn-shanghai.aliyuncs.com/machine-learning/Stanford/%E9%AB%98%E6%96%AF%E6%A0%B8%E5%87%BD%E6%95%B0%E9%A2%84%E6%B5%8B%E6%AD%A5%E9%AA%A42.png)

选择另一个不同的点，因为该点距离$l^{(1)}$、$l^{(2)}$、$l^{(3)}$都很远，所以$f_1$、$f_2$、$f_3$都接近于0。带入公式得：
$$
\theta_0+\theta_1f_1+\theta_2f_2+\theta_3f_3 = -0.5 \leq0
$$
所以对于这个新的选点，预测的$y=0$。

实际上，最后得到的结果是：**对于接近$l^{(1)}$、$l^{(2)}$的点，预测值是1，对于远离$l^{(1)}$、$l^{(2)}$的点，最后预测的结果是等于0的**。最后会得到这个预测函数的判别边界看起来是类似这样的结果：

![高斯核函数预测步骤3](http://gtw.oss-cn-shanghai.aliyuncs.com/machine-learning/Stanford/%E9%AB%98%E6%96%AF%E6%A0%B8%E5%87%BD%E6%95%B0%E9%A2%84%E6%B5%8B%E6%AD%A5%E9%AA%A43.png)

在这个红色的判别边界里面，预测的$y$值等于1；在这外面预测的$y$值等于0。因此这就是一个如何通过标记点以及核函数，来训练出非常复杂的非线性判别边界的方法。

#### 如何选取标记点(landmark)

假设数据集如下图，有一些正负样本，我们的想法是：直接将训练样本作为标记点，利用这个方法最终能得到m个标记点：

![svm选取标记点](http://gtw.oss-cn-shanghai.aliyuncs.com/machine-learning/Stanford/svm%E9%80%89%E5%8F%96%E6%A0%87%E8%AE%B0%E7%82%B9.png)

这m个标记点记作：$l^{(1)},l^{(2)},\ldots,l^{(m)}$，即$l^{(i)}=x^{(i)}$，特征函数基本上是在描述**每一个样本距离样本集中其他样本的距离**。

当输入样本$x$（样本$x$可以属于训练集，也可以属于交叉验证集，也可以属于测试集），然后可以计算这些特征：
$$
f_i=simlarity(x,l^{(i)})
$$
最终会得到一个特征向量$f$:
$$
f =
\begin{bmatrix}
   f_1  \\
   f_2  \\
   \ldots \\
   f_m
\end{bmatrix}
$$
按照惯例需要添加额外的特征$f_0$，且$f_0=1$
$$
f =
\begin{bmatrix}
   f_0  \\
   f_1  \\
   f_2  \\
   \ldots \\
   f_m
\end{bmatrix}
$$
对于训练样本$(x^{(i)},y^{(i)})$，对应的特征向量可以这样计算，即<mark>当前训练样本和所有样本的核函数</mark>：
$$
f_1^{(i)}=simlarity(x^{(i)},l^{(1)}) \\
f_2^{(i)}=simlarity(x^{(i)},l^{(2)}) \\
\cdots \\
f_m^{(i)}=simlarity(x^{(i)},l^{(m)})
$$
对于这个样本来说，其中的某一个特征等于1。接下来将这m个特征合并为一个特征向量。于是，相比之前用$x^{(i)}$来描述样本，$x^{(i)}$为n维（或n+1维）空间。现在可以使用这个特征向量$f^{(i)}$来描述特征向量(其中$ f_0^{(i)}$=1)：
$$
f^{(i)} =
\begin{bmatrix}
   f_0^{(i)}  \\
   f_1^{(i)}  \\
   f_2^{(i)}  \\
   \ldots \\
   f_m^{(i)}
\end{bmatrix}
$$

### 整合

#### 获取参数$\theta$

在使用SVM学习算法的时候，具体来说就是要求解这个最小化问题：
$$
min_\theta\ C\sum_{i=1}^{m}[y^{(i)}cost_1(\theta^Tf^{(i)}) + (1-y^{(i)})cost_0((\theta^Tf^{(i)})] + \frac{1}{2}\sum_{j=1}^{m}\theta_j^2
$$
通过解决这个最小化问题(把之前$x^{(i)}$换成$f^{(i)}$，$\sum_{j=1}^{n}\theta_j^2$中$n$也就成了$m$)，就能得到支持向量机的参数。

其实在实现支持向量机时，你并不需要知道这个细节，事实上这个式子已经提供了全部需要的原理。并不需要知道怎么去写一个软件，来最小化该代价函数，你能找到很好的且成熟的软件来做这些。

#### 做出预测

如果已经得到参数$\theta$并且想对样本$x$做出预测，先要计算特征向量$f$，<mark>如果样本$x$每个维度取值跨度很大，需要做均值归一化处理</mark>。$f$是$m+1$维的特征向量（这里有m是因为我们有m个训练样本，因此就有m个标记点）。

当$\theta^Tf \geq 0$时，预测$y=1$。

#### 参数选取

##### SVM参数

在使用支持向量机时，关于选择支持向量机中的参数$C=\frac{1}{\lambda}$，$C$对应着之前逻辑回归的的$\lambda$:

- 较小的$\lambda$对应较大的$C$，这就意味着有可能得到一个低偏差但高方差的模型。
- 较大的$\lambda$对应较小的$C$，这就意味着有可能得到一个高偏差但低方差的模型。

##### $\sigma^2$

- <mark>**如果$\sigma^2$越小**，那么高斯核函数会变化的**很剧烈**，在这种情况下，最终得到的模型会是**低偏差和高方差**</mark>。
- <mark>**如果$\sigma^2$越大**，那么高斯核函数倾向于变得**越平滑**，由于函数平滑且变化的比较平缓，这会给模型带来**较高的偏差和较低的方差**</mark>。

这就是利用核函数的支持向量机算法。

```matlab
% 计算出使误差最小的C和sigma
function [C, sigma] = dataset3Params(X, y, Xval, yval)

% return the following variables.
C = 1;
sigma = 0.3;

% ====================== YOUR CODE HERE ======================

list = [0.01, 0.03, 0.1, 0.3, 1, 3, 10, 30];
maxError = 1; % 假设模型完全不匹配，误差率为1

for i = 1 : length(list)
	for j = 1 : length(list)
		C_test = list(i);
		sigma_test = list(j);
		% 调用SVM学习算法，求参数theta
		model= svmTrain(X, y, C_test, @(x1, x2) gaussianKernel(x1, x2, sigma_test));
		% SVM模型预测出的y值
		predictions = svmPredict(model, Xval);
		% 求预测后的误差
		errors = mean(double(predictions ~= yval));
		% 选取使误差最小的C和sigma
		if errors <= maxError
			C = C_test;
			sigma = sigma_test;
			maxError = errors;
		end
	end
end

% 最后让值打印出来
C
sigma
maxError

% =========================================================================

end

```



### 关于模型选取

1. 如果相较于 m 而言,n 要大许多,即训练集数据量不够支持我们训练一个复杂的非线性模型,我们选用逻辑回归模型或者不带核函数的支持向量机
2. 如果 n 较小,而且 m 大小中等,例如 n 在 1-1000 之间,而 m 在 10-10000 之间,使用高斯核函数的支持向量机。
3. 如果 n 较小,而 m 较大,例如 n 在 1-1000 之间,而 m 大于 50000,则使用支持向量机会非常慢,解决方案是创造、增加更多的特征,然后使用逻辑回归或不带核函数的支持向量机。

神经网络在以上三种情况下都可能会有较好的表现,但是训练神经网络可能非常慢,选择支持向量机的原因主要在于它的代价函数是凸函数,不存在局部最小值。





