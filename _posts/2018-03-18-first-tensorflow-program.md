---
layout: post
title:  "第一个tensorflow程序"
date:   2018-03-18 22:39:00 +0800
tags: tensorflow
categories: 日常记录
---

使用docker镜像运行一个tensorflow的Hello World项目。


安装了ubuntu 18.04后，通过pip安装tensorflow总是莫名奇妙出错，只能祭出docker大法。用docker的话只要一个镜像就可以运行，没有其他依赖。


## docker安装tensorflow

### 1. 安装 docker

```
$ sudo apt install docker.io
```

### 2. 将用户加入到docker组中，这样不需要sudo就可以运行docker镜像

```
$ sudo usermod -a -G docker $HOME
$ sudo systemctl restart docker
```

### 3. 测试安装成功

```
$ docker run hello-world
```

### 4. docker使用代理(可选)

如果因为网络问题，无法通过docker pull拉取镜像，可尝试使用代理。
docker使用代理的方式见[官方文档](https://docs.docker.com/config/daemon/systemd/#httphttps-proxy)

* 为docker服务创建一个内嵌的systemd目录

```
$ mkdir -p /etc/systemd/system/docker.service.d
```

* 创建`/etc/systemd/system/docker.service.d/http-proxy.conf`文件，并添加`HTTP_PROXY`环境变量。其中[proxy-addr]和[proxy-port]分别改成实际情况的代理地址和端口：

```
[Service]
Environment="HTTP_PROXY=http://[proxy-addr]:[proxy-port]/" "HTTPS_PROXY=https://[proxy-addr]:[proxy-port]/"
```

* 如果还有内部的不需要使用代理来访问的Docker registries，需要指定NO_PROXY环境变量：

```
[Service]
Environment="HTTP_PROXY=http://[proxy-addr]:[proxy-port]/" "HTTPS_PROXY=https://[proxy-addr]:[proxy-port]/" "NO_PROXY=localhost,127.0.0.1,docker-registry.somecorporation.com"
```

* 更新配置：

```
$ systemctl daemon-reload
```

* 重启Docker服务：

```
$ systemctl restart docker
```

### 5. 拉取tensorflow镜像

```
$ docker pull tensorflow/tensorflow
```

这里拉取镜像可能时间会比较长。

## 利用MNIST数据集识别手写图片

当我们开始学习编程的时候，第一件事往往是学习打印"Hello World"。就好比编程入门有Hello World，机器学习入门有MNIST。MNIST是一个入门级的计算机视觉数据集，它包含各种手写数字图片。这里将训练一个机器学习模型用于预测图片里面的数字。此范例来自[MNIST机器学习入门](http://wiki.jikexueyuan.com/project/tensorflow-zh/tutorials/mnist_beginners.html)。

* 拉取MNIST数据集，包括了训练模型的数据和测试数据集，镜像中的已经安装了inout_data模块，调用read_data_sets会自动下载导入数据集)。

```python
import tensorflow.examples.tutorials.mnist.input_data as input_data
mnist = input_data.read_data_sets("MNIST_data/", one_hot=True)
```

* 导入tensorflow包

```python
import tensorflow as tf
```

* 定义x和y_两个占位符，之后的tensorflow通过这两个占位符输入正确数。x代表图片信息，因为图片为28x28像素(28x28=784)，None表示任意数量。y_表示图片代表的数字，有0-9共10个数字。

```python
x = tf.placeholder(tf.float32, [None, 784])
y_ = tf.placeholder("float", [None,10])
```

* 定义softmax回归模型，W和b是两个变量

```python
W = tf.Variable(tf.zeros([784,10]))
b = tf.Variable(tf.zeros([10]))
y = tf.nn.softmax(tf.matmul(x,W) + b)
```

* 通过梯度下降算法训练模型。cross_entropy表示交叉熵，这里用来衡量模型的误差，交叉熵越小表示模型与训练数据的误差越小。y_为输入的实际分布，y为预测的概率分布。 `GradientDescentOptimizer`如果了解机器学习算法的话应该很熟悉， 表示用梯度下降算法进行训练，学习速率为0.01，目的是使cross_entropy最小。

```python
cross_entropy = -tf.reduce_sum(y_*tf.log(y))
train_step = tf.train.GradientDescentOptimizer(0.01).minimize(cross_entropy)
```

* 启动模型，每次通过mnist.train.next_batch从数据集中取100条数据进行模型训练，循环1000次

```python
init = tf.initialize_all_variables()
sess = tf.Session()
sess.run(init)

for i in range(1000):
    batch_xs, batch_ys = mnist.train.next_batch(100)
    sess.run(train_step, feed_dict={x: batch_xs, y_: batch_ys})
```

* 通过MNIST的测试集测试模型识别的准确率

```python
correct_prediction = tf.equal(tf.argmax(y,1), tf.argmax(y_,1))
accuracy = tf.reduce_mean(tf.cast(correct_prediction, "float"))
print sess.run(accuracy, feed_dict={x: mnist.test.images, y_: mnist.test.labels})
```

## docker中运行

* 在本机新建一个文件夹，这里命名tensorflow，将上面的代码写入到文件夹下的test.py中

```
$ mkdir tensorflow
$ vim tensorflow/test.py
```

* 启动docker，将tensorflow文件夹以虚拟卷方式挂载到docker实例中

```
$ docker run -it -v $PWD/tensorflow:/tensorflow tensorflow/tensorflow /bin/bash
```

* 此时当前终端在docker实例中，通过`python test.py`运行

```
# cd /tensorflow
# python test.py
```

识别的准确率在91%左右。可以试着减少或者增加模型训练的循环次数，再比较识别的准确率。可以发现循环超过100次后，仅靠增加循环字数并不能增加识别的准确率。