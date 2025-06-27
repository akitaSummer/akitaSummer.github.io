---
title: 从线性回归到MLP
date: 2025-01-20 00:21:35
tags:
  - machine learning
  - python
categories: 学习笔记
---

# 线性模型
## 模型搭建
线性模型是最简单的数学模型，他的数学公式是下面这个:
$$f(\vec{x})=\sum_{i=1}^{n}w_{i}x_{i}+b$$
权重＋变量求和最后加上常量。

我们以一个地区房价的变动预测为例，他的参数可能有成本，利润，人数，收入等等可能与预测值有关系的参数，这些统计就是在做特征工程。

我们收集的数据可以表示为
$$X=[\vec{X}_{1},\vec{X}_{2},\vec{X}_{3},\ldots\vec{X}_{N}]^{T}$$
$$Y=[\vec{Y}_{1},\vec{Y}_{2},\vec{Y}_{3},\ldots\vec{Y}_{N}]^{T}$$
其中X是特征计算，Y是房价，N是数据数目。

接下来我们就是训练，通过调整参数使得根据 $x$ 计算出来的 $f(\vec{x})$ 与真实的 $Y$ 尽可能的差距小。

这里有个需要注意的地方，我们知道房价的值是不能小于0的，这时候我们需要对模型进行转化 (在深度学习里叫激活函数) ：
$$f(\overrightarrow{x})=\max(0,\sum_{i=1}^{n}w_{i}x_{i}+b)$$
其中这个 $\max(x,y)$ 又叫 [ReLU 函数](https://zhuanlan.zhihu.com/p/428448728)，当然也有别的函数，比如结果只在 0~1 之间的 [Sigmoid 函数](https://zhuanlan.zhihu.com/p/424858561)等等，这里就不多做介绍了

## 损失函数
损失函数通常用于计算模型对单个样本预测结果的误差，房价例子中的损失函数可以表示为: 
$$C(\vec{w},b,X,Y)=\frac{1}{N}\sum_{i=1}^{N}(y_{i}-\max(0,f(\vec{x}_{i},\vec{w},b)))^{2}$$
我们通过计算模型使用的 ReLU 函数计算出的结果，与真实值相减的平方求平均数，让他尽可能的小，通过这种形式，讲一个优化问题，转换为了数学问题。

现在，我们的目标就变成了，调整 $\vec{w}$ 和 $b$ ，让损失函数结果尽可能小。

这里会引出一个经典问题，过拟合，因为学习实际上他并不是一个真实的数学优化问题，如果过拟合，会导致泛化能力变弱，这是需要注意的，这也是机器学习和数学优化的重要区别。

![Cost function](/image/machine-learning/cost-funtion.png)

## 如何优化
这里推荐 Stephen Boyd 的 [Convex Optimization](https://www.youtube.com/watch?v=kV1ru-Inzl4)，这里只做一些简单的介绍，一般优化分为两种，一种是梯度性优化，另一种是对偶性优化（一般是转成更好优化的问题去优化）。这里我们主要介绍在我们这个模型中用到的梯度性下降优化。

梯度就是这个函数的不同变量倒数组成的变量。

$$[\frac{\partial f}{\partial x_{1}},\frac{\partial f}{\partial x_{2}}\cdots\frac{\partial f}{\partial x_{n}}]$$

梯度下降你可以理解为，我们有个函数，我们通过寻找他变量中各个方向上下降最快的方向去移动，从而拿到他的全局最低点。

![optimization](/image/machine-learning/optimization.jpg)

比如左图，当导数为0时，可以拿到他的最小值，这个优化的前提是这个函数是凸函数。

如果出现右图这种不是凸函数，我们可能需要通过修改步长（随机扰动），从而跳过局部最优解，找到全局最优解。非凸最优解问题在数学上是一个比较复杂的事情，我们可以招一个近似最优解来解决。

## 链式法则
知道了优化方法后，我们下一步就是要求梯度，也就是求偏导。这里介绍一下大学高数里的链式法则，他可以把复杂的求导化简。

$$y(x)=f(g(x))$$

$$y^{\prime}(x)=f^{\prime}(g(x))\cdot g^{\prime}(x)$$

这里帮大家回忆一下这个证明
$$y^{\prime}(x)=\lim_{\Delta x\to0}\frac{f(g(x+\Delta x))-f(g(x))}{\Delta x}$$
$$=\lim_{\Delta x\to0}\frac{f(g(x)+\Delta y)-f(g(x))}{\Delta x}$$
$$=\lim_{\Delta x\to0}\frac{f(g(x)+\Delta g)-f(g(x))}{\Delta g}\cdot\frac{\Delta g}{\Delta x}$$
$$=f^{\prime}(g(x))\cdot g^{\prime}(x)$$
$$let:\Delta g=g(x+\Delta x)-g(x))$$
$$g(x+\Delta x)=g(x)+\Delta g$$

有了这个后，我们可以看链式法则在向量上的推广，假设我们有个方程组:
$$\vec{f}(\vec{x})\Rightarrow\begin{cases}
f_{1}(\overrightarrow{x})=ax_{1}+bx_{2}^{2}+cx_{3}^{3} \\
f_{2}(\overrightarrow{x})=dx_{1}+\sin x_{2} \\
f_{2}(\overrightarrow{x})=\cdots x_{1}+\cdots x_{2} & 
\end{cases}$$

他的其中一个函数的梯度就是:
$$[\frac{\partial f_{1}}{\partial x_{1}},\frac{\partial f_{1}}{\partial x_{1}},\frac{\partial f_{1}}{\partial x_{3}}]$$

整体梯度 Jacobi 矩阵就是:
$$[
\begin{array}
{c}\frac{\partial f_{1}}{\partial x_{1}},\frac{\partial f_{1}}{\partial x_{2}},\frac{\partial f_{1}}{\partial x_{3}} \\
\frac{\partial f_{2}}{\partial x_{1}},\frac{\partial f_{2}}{\partial x_{2}},\frac{\partial f_{2}}{\partial x_{3}} \\
\frac{\partial f_{3}}{\partial x_{1}},\frac{\partial f_{3}}{\partial x_{1}},\frac{\partial f_{3}}{\partial x_{3}}
\end{array}]$$
中间的计算，可以参考 [The Matrix Calculus You Need For Deep Learning](https://explained.ai/matrix-calculus/) 这本小书，里面介绍了深度学习需要的矩阵计算的知识。结果就是：
$$\frac{\partial \vec{f}(\overrightarrow{g}(\overrightarrow{x}))}{\partial \overrightarrow{x}}=\frac{\partial \overrightarrow{f}}{\partial \overrightarrow{g}}\cdot\frac{\partial \overrightarrow{g}}{\partial \overrightarrow{x}}$$
其中
$$f(\overrightarrow{x})\Rightarrow
\begin{cases}
g_{1}(\overrightarrow{x}) \\
g_{2}(g_{1}(\overrightarrow{x})) \\
g_{3}(g_{2}(\overrightarrow{x})) & 
\end{cases}$$
可以得到
$$\frac{\partial f(\overrightarrow{x})}{\partial\overrightarrow{x}}=\frac{\partial g_{3}}{\partial g_{2}}\cdot\frac{\partial g_{2}}{\partial g_{1}}\cdot\frac{\partial g_{1}}{\partial\overrightarrow{x}}$$

## 解决问题
有了以上知识，我们可以回到我们之前的房价问题上来解决它了。我们的需求是将损失函数最小化：
$$C(\vec{w},b,X,Y)=\frac{1}{N}\sum_{i=1}^{N}(y_{i}-\max(0,f(\vec{x}_{i},\vec{w},b)))^{2}$$
我们可以将损失函数转化为
$$C(\vec{w},b,X,Y)\Rightarrow\begin{cases}
u(\overrightarrow{w},\overrightarrow{x},b) & =\max(0,\overrightarrow{w}\overrightarrow{x}_{i}+b) \\
v(y,u) & =y-u \\
c(v) & =\frac{1}{N}\sum_{i=1}^{N}v^{2} & 
\end{cases}$$
接下来我们一层一层求偏导
$$\frac{\partial u}{\partial\overrightarrow{w}}=
\begin{cases}
\overrightarrow{0}^{T} & \overrightarrow{w}\overrightarrow{x_{i}}+b\leq0 \\
\overrightarrow{x} & \overrightarrow{w}\overrightarrow{x_{i}}+b>0 & 
\end{cases}$$
$$\frac{\partial v}{\partial\vec{w}}=\frac{\partial v}{\partial u}\cdot\frac{\partial u}{\partial\vec{w}}=(0-1)\frac{\partial u}{\partial\vec{w}}=-\frac{\partial u}{\partial\vec{w}}$$
$$\frac{\partial c}{\partial\vec{w}}=\frac{\partial}{\partial\vec{w}}\frac{1}{N}\sum_{i=1}^{N}v^{2}=\frac{1}{N}\sum_{i=1}^{N}\frac{\partial}{\partial\vec{w}}v^{2}=\frac{1}{N}\sum_{i=1}^{N}2v\frac{\partial v}{\partial\vec{w}}$$

到了最外层后，我们就可以把它们整合起来了
$$\frac{\partial C}{\partial\vec{w}}=\frac{\partial}{\partial\vec{w}}\frac{1}{N}\sum_{i=1}^{N}v^{2}=\frac{1}{N}\sum_{i=1}^{N}\frac{\partial}{\partial\vec{w}}v^{2}=\frac{1}{N}\sum_{i=1}^{N}2v\frac{\partial v}{\partial\vec{w}}=\frac{1}{N}\sum_{i=1}^{N}
\begin{cases}
\vec{0}^{T},(\vec{w}\vec{x}_{i}+b\leq0) \\
-2v\vec{x}^{T}(\vec{w}\vec{x}_{i}+b>0) & 
\end{cases}=
\begin{cases}
\vec{0}^{T},(\vec{w}\vec{x}_{i}+b\leq0) \\
\frac{2}{N}\sum_{i=1}^{N}(\overrightarrow{w}\cdot\overrightarrow{x}_{i}+b-y_{i})x_{i}^{T} & 
\end{cases}=
\begin{cases}
\vec{0}^{T},(\vec{w}\vec{x}_{i}+b\leq0) \\
\frac{2}{N}\sum_{i=1}^{N}e_{i}x_{i}^{T} & 
\end{cases}$$

其中 $e_{i} = \overrightarrow{w}\cdot\overrightarrow{x}_{i}+b-y_{i}$ 是误差函数

这样就可以得到我们的算法了：
- 我们先随机取一个随机权重 $\vec{w}_{0}^{T}$ ，查看结果，如果打到可接受范围则结束，否则下一步
- 计算梯度 $\frac{2}{N}\sum_{i=1}^{N}e_{i}x_{i}^{T}$
- 得到新权重 $\vec{w}_{1}^{T} = \vec{w}_{0}^{T} - \frac{2}{N}\sum_{i=1}^{N}e_{i}x_{i}^{T}\cdot\eta$

## Coding
```python
import numpy as np

class MyLinearRegression:
    # 学习率 迭代次数 激活函数
    def __init__(self, learning_rate=0.01, n_iterations=1000, activation=None):
        self.learning_rate = np.float64(learning_rate)
        self.n_iterations = n_iterations
        self.weights = None
        self.bias = None
        self.activation = activation
    
    def _relu(self, x):
        return np.maximum(0, x)

    def _relu_derivative(self, x):
        return (x > 0).astype(float)

    def _sigmoid(self, x):
        return 1 / (1 + np.exp(-x))

    def _sigmoid_derivative(self, x):
        s = self._sigmoid(x)
        return s * (1 - s)
    
    def _apply_activation(self, x):
        if self.activation == 'relu':
            return self.relu(x)
        elif self.activation == 'sigmoid':
            return self.sigmoid(x)
        return x  # No activation

    def _apply_activation_derivative(self, x):
        if self.activation == 'relu':
            return self.relu_derivative(x)
        elif self.activation == 'sigmoid':
            return self.sigmoid_derivative(x)
        else:
            return 1 # Derivative is 1 when no activation

    def fix(self, X, y):
        n_samples, n_features = X.shape

        # 初始随机权重
        self.weights = np.random.randn(n_features).astype(np.float64) * 0.01
        self.bias = np.float64(0)

        # 退出阈值，连续五次小于 1e-5
        prev_loss = float('inf')
        patience = 5
        min_change = 1e-5
        patience_counter = 0

        for _ in range(self.n_iterations):
            linear_output = np.dot(X, self.weights) + self.bias
            y_predicted = self.apply_activation(linear_output)
            
            activation_derivative = self.apply_activation_derivative(linear_output)
            diff = y_predicted - y

            # w 和 b 的梯度
            dw = np.float64(1/n_samples) * np.dot(X.T, (diff * activation_derivative))
            db = np.float64(1/n_samples) * np.sum(diff * activation_derivative)

            # 设置个范围，不要变化过大
            clip_threshold = 2.0
            dw = np.clip(dw, -clip_threshold, clip_threshold)
            db = np.clip(db, -clip_threshold, clip_threshold)

            if not (np.isnan(dw).any() or np.isnan(db)):  
                self.weights -= self.learning_rate * dw
                self.bias -= self.learning_rate * db

            current_loss = np.mean(y_predicted - y) ** 2
            # 小于阈值就结束
            if abs(prev_loss - current_loss) < min_change:
                patience_counter += 1
                if patience_counter >= patience:
                    break
            else:
                patience_counter = 0
            prev_loss = current_loss
    
    def predict(self, X):
        linear_output = np.dot(X, self.weights) + self.bias
        return self.apply_activation(linear_output)

    
    def score(self, X, y):
        y_pred = self.predict(X)
        ss_total = np.sum((y - y.mean()) ** 2)
        ss_residual = np.sum((y - y_pred) ** 2)
        return 1 - (ss_residual / ss_total)
```

```python
# 计算用
import numpy as np
# 数据分析用
import pands as pd

from sklearn.model_selection import train_test_split
from sklearn.linear_model import LinearRegression
from LinearModel import MyLinearRegression

if __name__ == '__main__':
    # load data
    df = pd.read_csv('California_Houses.csv')

    # 拆分
    X = df.drop('Median_House_Value', axis=1)
    y = df.drop['Median_House_Value']

    # 训练集，测试集
    X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

    # 先用别人的模型跑下
    model = LinearRegression()
    model.fit(X_train, y_train)
    print("R-squared score: ", model.score(X_test, y_test))

    myModel = MyLinearRegression(learning_rate=0.001, n_iterations=10000, activation='relu')
    myModel.fit(X_train, y_train)
    print("R-squared score: ", model.score(X_test, y_test))

    # 尝试优化下模型
    # 先看下分布图
    # numerical_columns_columns_coltypes(include=[`float64',"int64']).columns
    # num_columns_len(numerical_columns)
    # num_rows = (num_columns // 3)+(1 if num_columns % 3 !=0 else 0)
    # num_cols = 3

    # plt.figure(figsize=(15, 6 * num_rows))
    # for i, column in enumerate(num_columns, 1):
    #     plt.subplot(num_rows, num_cols, i)
    #     sns.histplot(df[column], kde=True)
    #     plt.title(f'Distribution of {column}')
    # plt.tight_layout()
    # plt.show()


    df['Distance_to_BigCty'] = np.min(df['Distance_to_LA','Distance_to_SanDiego'])

    df['Bedroom_per_household'] = df['Households'] / df['Households']

    df['Median_House_Value'] = np.log(df['Median_House_Value'])
    df['Median_Income'] = np.log(df['Median_Income'])
    df['Distance_to_coast'] = np.log(df['Distance_to_coast'])
    df['Bedroom_per_household'] = np.log(df['Bedroom_per_household'])

    newX = df.drop('Median_House_Value', axis=1)
    new_y = df.drop['Median_House_Value']

    # 训练集，测试集
    new_X_train, new_X_test, new_y_train, new_y_test = train_test_split(newX, new_y, test_size=0.2, random_state=42)

    model = LinearRegression()
    model.fit(new_X_train, new_y_train)
    print("R-squared score: ", model.score(new_X_test, new_y_test))
```
<!-- 
# 决策树

假设我们有个贷款是否能收回的一个场景，我们有以下数据

$$x=[employed,revenue,student,\ldots,age]^{T}$$

$$X=[\vec{x}_{1},\vec{x}_{2},\vec{x}_{3},\ldots\vec{x}_{n}]^{T}$$

$$Y=[\vec{y}_{1},\vec{y}_{2},\vec{y}_{3},\ldots\vec{y}_{n}]^{T}$$

通过训练一棵树，来预测用户贷款是否能还款，即使 $f(\vec{x})$ 尽量逼近 $Y$ 。

![Decision Tree](/image/machine-learning/decision-tree.png)

与线性模型不同的点在于，每一个节点如何构建下一个节点跟线性模型调权重有非常大的不同。


## 熵和信息增益

熵是用来描述系统的混乱程度，即不确定性。在机器学期中分类中说，熵越大即这个类别的不确定性更大，反之越小。当 p=0 或 p=1 时，H(p)=0,随机变量完全没有不确定性，当p=0.5时，H(p)=1,此时随机变量的不确定性最大

![entropy](/image/machine-learning/entropy.png)

$$H(X)=\sum_{i=1}^{n}p(x_{i})\:I(x_{i})=-\sum_{i=1}^{n}p(x_{i})\log_{2}p(x_{i})$$

### 哈弗曼编码

假设我们的哈弗曼编码只有四个字母，A(00), B(01), C(10), D(11)，假设，每个字母的出现率是 P(A)= 50%  P(B)= 25%  P(C)= 12.5%  P(D)= 12.5%，则这个系统的熵为 H = 1.75，而我们用了 2bit 来表示。

但是如果我们的编码用的是 A(0), B(10), C(110), D(111)，那他的平均 bit = 1 * 0.5 + 2 * 0.25 + 3 * 0.125 + 3 * 0.125 = 1.75 bit

这里例子表示了，如果系统的熵越低，实际上编码数越低。

### 信息增益

通过刚才的例子我们需要做的是，通过不断地增加节点，让子节点的熵降低，从而表示这个树信息越多。 -->

# MLP
刚才所讲的线性模型，如果用图画出来就是这样

![Linear model](/image/machine-learning/linear-model.png)

数学公式是下面这个:
$$f(\vec{x})=\sum_{i=1}^{n}w_{i}x_{i}+b$$

当我们把很多线性模型，添加激活函数，组合在一起，就成了一个多层感知机，也就是MLP

![MLP](/image/machine-learning/MLP.png)


数学公式是下面这个:

$$f(\vec{x})=\sum_{j=1}^{n}activation_{j}(\sum_{i=1}^{n}w_{i}x_{i}+b)$$

其中的activation就是我们的激活函数，可以使RELU，或者sigmoid等等，如果没有激活函数，那么MLP就是一个超大型的线性函数而已，正是因为激活函数，才能让他拥有魔法。

推荐一个🌰，[识别手写数字](https://www.bilibili.com/video/BV1bx411M7Zx/?spm_id_from=333.1387.collection.video_card.click&vd_source=1c9c4de6b1ffda02913fc889a72af206)。

## 建立数学模型
参考视频，我们要将这个模型先转化为数学问题，他有4层节点，我们可以从左到右设为$\overrightarrow{y_{0}}$,$\overrightarrow{y_{1}}$,$\overrightarrow{y_{2}}$,$\overrightarrow{y_{3}}$

我们选用Sigmoid作为激活函数，他的数学描述为：

$$\begin{cases}
\vec{y}_3 = Sigmoid \left( \vec{w}_2 \cdot \vec{y}_2 + \vec{b}_2 \right) \\
\vec{y}_2 = Sigmoid \left( \vec{w}_1 \cdot \vec{y}_1 + \vec{b}_1 \right) \\
\vec{y}_1 = Sigmoid \left( \vec{w}_0 \cdot \vec{y}_0 + \vec{b}_0 \right) & 
\end{cases}$$

接下来我们需要优化这个数学模型，因此我们需要一个损失函数

$$cost = (\overrightarrow{y}_3 - \overrightarrow{y})^2$$

我们要通过调整$\overrightarrow{w}/\overrightarrow{b}$来使最终的损失函数达到最小

我们先回到线性模型中，如果只有一个简单的线性方程$y=Sigmoid(w\cdot x + b)$，那我们其实只用对w和b求偏导，但是我们这里有$\vec{w}_2$,$\vec{w}_1$,$\vec{w}_0$,$\vec{b}_2$,$\vec{b}_1$,$\vec{b}_0$六个变量，这个时候就要用到反向传播

## 反向传播
反向传播实际上就是基于链式法则，在刚在的例子中，我们以最终损失函为例，将其可拆分为

$$\begin{cases}
C=\frac{1}{N}\sum_{i=1}^{N}\vec{v} \\
\vec{v} = (\overrightarrow{y}_3 - \overrightarrow{y})^2 \\
\vec{y}_3 = Sigmoid(\vec{a}_3) \\
\vec{a}_3 = (\vec{w}_2 \cdot \vec{y}_2 + \vec{b}_2) \\
...
\end{cases}$$

当我们求出$[\frac{\partial C}{\partial \vec{w}_{0}},\frac{\partial C}{\partial \vec{w}_{1}},\frac{\partial C}{\partial \vec{w}_{2}},\frac{\partial C}{\partial \vec{b}_{0}},\frac{\partial C}{\partial \vec{b}_{1}},\frac{\partial C}{\partial \vec{b}_{2}}]$梯度后，我们就可以使用梯度下降来不断地训练