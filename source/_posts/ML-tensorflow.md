---
title: 小试ML牛刀---Tensorflow预测房价
date: 2017-07-09 00:32:30
tags:
    - Machine Learning
    - TensorFlow
---

### 零、ML(Machine Learning)到底是嘛？

Tom Mitchell于1998年说道：

> A computer program is said to learn from experience E with respect to some task T and some performance measure P, if its performance on T as measured by P, improves with experiance E.

计算机程序根据现有的经验能够完成某种任务，达到一定的性能指标，并且随着经验的增多，能够不断的提升性能。

机器学习根据训练的样本是否有label又分为监督学习、非监督学习。
#### 监督学习
对给定的训练样本，已知其正确的label，通过对数据的学习，能够尽可能的预测样本以外的数据，监督学习一般又根据需要预测的值是连续还是离散的，分为回归问题和分类问题
#### 非监督学习
给定的训练样本，没有任何label信息，而非监督学习就是要在这些数据中寻找某些模式，将其分成不同的类别，Google News就使用无监督学习对新闻进行分类

###  一、TensorFlow是什么？

TensorFlow是一个采用数据流图，用于数值计算的开源软件库，主要用于机器学习和深度学习，由Google Brain开发，于2015年9月开源。
####  计算图（computational graph)
TensorFlow是基于计算图的框架，在具体介绍TensorFlow之前，我们先看看什么是计算图。假设我们有一个需要计算的表达式：y = a * b + c，该表达式包含一个乘法和一个加法，该表达式可以表示为：

上图完整描述了计算任务的依赖关系，这种有向无环图就叫做计算图。

在TensorFlow中主要有以下几个概念：
####  Tensor
Tensor（”张量“）是TensorFlow中最重要的数据单元，一个tensor由多维数组构成。
tensor的rank表示其数组的维度，tensor的shape代表每个纬度数据的个数，例如：
```
3 # 0维张量，即标量, rank = 0, shape = 1
[1. ,2., 3.] # 1维张量，即向量, rank = 1, shape = 3
[[1., 2., 3.,4.], [5., 6., 7.,8.]] # 2维张量， 一个2 x 3 的矩阵, rank = 2, shape =3
```

####  Operation
执行计算的单元，可以是加减乘除等数学运算，也可以是各种各样的优化算法，Operation接收0个或多个Tensor，输出一个Tensor
####  Node
在计算图中用于代表Tensor或者Operation
#### Graph
Graph用于定义整个计算任务，没有进行任何计算，在TensorFlow中，可以通过查看session.graph_def来得到graph的定义：
```javascript
node {
  name: "a/initial_value"
  op: "Const"
  attr {
    key: "dtype"
    value {
      type: DT_INT32
    }
  }
  attr {
    key: "value"
    value {
      tensor {
        dtype: DT_INT32
        tensor_shape {
        }
        int_val: 1
      }
    }
  }
}
...
```
#### Session
Graph仅仅定义了所有的Node，没有进行任何计算，而session根据graph的定义分配资源，执行计算任务。


###  二、安装
TensorFlow的安装非常简单，有兴趣的同学可以参考[官网](https://www.tensorflow.org/install/)，笔者这里使用pip在mac上面进行安装，遇到一个小坑，安装TensorFlow时出现了[Errno 1] Operation not permitted，遇到相同问题的可以移步[这里](https://stackoverflow.com/questions/32659348/operation-not-permitted-when-on-root-el-capitan-rootless-disabled/32661637#32661637)(官网也有关于这个问题的link，但是并不能解决问题)

### 三、TensorFlow的使用
假设现在我们有一组Data Set表示房屋的面积以及对应房屋的价格，我们想预测面积为90的房屋价格应该是多少，DataSet如下：

|X|Y|
|---|---|
|40|37.0000|
|65|72.0164|
|80|93.0456|
|115|130.4864|
|150|153.4161|

其中X为房屋的面积，Y为房屋的价格，对于Tom给出的机器学习定义来说，这里的E就是已有的房屋价格数据，T就是对房屋价格进行预测，P就是房屋价格预测的准确性。
先在坐标系中绘制房屋价格数据如下：

观察上图，几乎是一条直线，可以使用linear regression来处理这个问题，来对这些数据进行学习，获得最优的θ<sub>0</sub>和θ<sub>1</sub>，使得平方误差函数的值最小
###### 拟合函数
h<sub>0</sub> = θ<sub>0</sub> + θ<sub>1</sub> * x
###### 参数
θ<sub>0</sub>, θ<sub>1</sub>

###### 损失函数：

其中m为数据集的大小，x<sup>i</sup>表示第i个数据集输入，y<sup>i</sup>表示对应的正确房价。

当使用TensorFlow构建graph时，大致分为5个部分：
1. 为输入x(对于多特征问题有多个输入)与输出y定义placeholder
2. 定义参数
3. 定义模型结构
4. 定义损失函数
5. 定义优化算法

####  代码
#####  定义placeholers与Variables
```python
import tensorflow as tf

# input
x = tf.placeholder(tf.float32, name="x")
# output
y = tf.placeholder(tf.float32, name="y")

# parameter
theta0 = tf.Variable(.5, name="theta0")
theta1 = tf.Variable(.5, name="theta1")
```
#####  定义模型：
```python
hypothesis = theta0 + theta1 * x
```
使用平方误差函数计算损失：
```python
squared_delta = tf.square(hypothesis - y)
loss = tf.reduce_sum(squared_delta) / (2 * 5)
```
#####  定义优化器：
这里使用梯度下降算法，学习率设定为0.00001，梯度下降算法会不断的更新θ<sub>0</sub>和θ<sub>1</sub>的值，使损失变小。
*learning rate的选取不宜过大或者过小，过大可能导致损失函数无法收敛，过小导致循环的次数增多*

```python
optimizer = tf.train.GradientDescentOptimizer(learning_rate=0.00001)
training = optimizer.minimize(loss)
```
#####  进行训练：
```python
dataSet = {x: [40, 65, 80, 115, 150], y: [37.0000, 72.0164, 93.0456, 130.4864, 153.4161]}
session = tf.Session()
init = tf.global_variables_initializer()
session.run(init)

for i in range(1000):
    session.run(training, dataSet)
    print(session.run([theta0, theta1]))
    print(session.run(loss, dataSet))
```

训练1000次之后，得到θ<sub>0</sub>= 0.50757158, θ<sub>1</sub> = 1.0718392, 对比之前的θ<sub>0</sub>= 0.5, θ<sub>1</sub> =  0.5，如图：

可以看出，在训练1000次之后，得到的拟合函数明显优于之前最初的函数，将面积90代入拟合函数，得到价格96.97309958，预测90平米房屋售价约等于97万。

### 总结
本文简单介绍了TensorFlow的一些基本概念，并通过实现一个简单的房价预测实例介绍了TensorFlow的基本使用，具体感兴趣的同学可以移步[官网](https://www.tensorflow.org)