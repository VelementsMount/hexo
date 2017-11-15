---
title: Prometheus基础（三）
date: 2017/9/1 08:00:00
toc: true
tags: monitor
keywords: 
  - container
  - prometheus
  - monitor
categories: 容器监控
---

## prometheus的架构
![architecture.svg](http://ovqidxh71.bkt.clouddn.com/architecture.svg)
prometheus从目标主机或者从push-gateway拉取数据，存在本地文件系统并根据recordRules配置文件计算聚合数据或者根据alertRules计算是否发送预警。grafana可以直接使用prometheus提供的api发送promQL查询时序数据。

## prometheus的数据模型
**prometheus**将相同的 metrics(指标名称) 和 labels(一个或多个标签) 组成一条时间序列

**[metrics](https://prometheus.io/docs/concepts/data_model/)**一般是给监测对象起一个名字。

**[labels](https://prometheus.io/docs/concepts/data_model/)**一般是给监测对象提供一些额外信息的键值对，对一条时间序列不同维度的识别，promQL将通过这些标签很容易的过滤和聚合这些时间序列数据。

```
api\_http\_requests_total{method="POST", handler="/messages"}
```
存入数据库中时还会自动为它添加一个时间戳标记，所以一个时序序列是大量不同时间的相同指标相同标签的数据集合。

如果以传统数据库的理解来看这条语句，则可以考虑 http_requests_total是表名，标签是字段，而timestamp是主键，还有一个float64字段是值了。（Prometheus里面所有值都是按float64存储）

## Prometheus的指标类型
Prometheus的客户端软件包中提供了4中核心的指标类型，这四中类型仅仅在客户端存在区别，在服务端存储时转换为无类型的时间序列。

* **Counter**

累加器或者称作计数器，统计的指标值只能增加，不能减少，增加值不一定为1，可以用于请求的总数、访问时间的总和。

* **Guage**

指数器，指示当前统计的指标值的大小，值可以增大也可以减小，主要用户统计当前cpu的温度、最近一次访问的耗时。

* **Histogram**

直方图，统计指标值分散在不同区间的个数。相当于针对Guage做了一次再加工，统计的时候就将分散在不同区间的个数统计好了。比如统计每次访问耗时的数据分布情况，用Histogram可以统计小于200ms的访问次数，小于300毫秒的次数，小于500毫秒的次数……

* **Summary**

概述，它的作用我们通过一个例子来说明：比如我们监测的指标值为每次请求的响应时间，用Summary可以统计5min内95%的请求的响应平均用时，5min内80%的请求的响应用时……。我们也可以统计10min内60%的请求的响应的平均用时……其实Summary也是针对Counter或者Guage做的再次加工，只是在记录到数据库之前它计算好了再存入数据库。它和Histogram针对同一监测指标的区别是Summary将次数作为横坐标，Histogram是将次数作为纵坐标。

通过上面的介绍我们知道最基本的类型其实就是Counter和Guage两种，其他的类型都是在它们基础上的再加工。了解了这4中类型，我们才能选择正确的类型统计我们需要监测的数据

## promQL基础

我们使用过sql，我们都能感受到sql语言查询功能的强大之处，promQL的查询功能也异常强大，它也具有运算、过滤、分组等功能，它还提供大量的内置函数，可以让我们更加容易对时序数据进行操作。当我们学会了promQL我们就能很顺利的将我们想要展示的数据按照我们的要求完美的展示出来。

promQL是非常简单的，在开始学习promQL之前我们先看一些简单的promQL查询表达式：

```
http_requests_total
```
返回监测指标名为http_requests_total时间戳为当前时间的时序序列（它们的标签可能不同，所以结果可能有多条）

```
http_requests_total{job="apiserver", handler="/api/comments"}
```
返回标签job="apiserver", handler="/api/comments"，监测指标名为http_requests_total时间戳为当前时间的时序序列。大括号相当于sql的where

```
http_requests_total{job="apiserver", handler="/api/comments"}[5m]
```
返回5分钟前到当前时间的标签为job="apiserver", handler="/api/comments"，监测指标名为http_requests_total的时序序列。中括号相当于对时间加了一个维度的限制。

```
http_requests_total{job=~"server$"}
```
返回标签job的值为以server结尾的指标名为http_requests_total的时序序列。=~后面接一个正则表达式，表示此标签的值匹配后面的正则表达式，那么!~就是不匹配后面的正则

```
http_requests_total offset 5m
```
返回的是5分钟之前的请求总数

下面会介绍4个函数的使用

```
rate(http_requests_total[5m])
```
http_requests_total记录的是请求的总数的时序序列，http_requests_total[5m]记录的是5分钟内请求的总数的时序序列，rate是计算平均率的函数既计算每秒的平均增加数。所以这个promQL的作用是计算5分钟内平均每秒的请求数。如果我们记录指标有

```
http_requests_total{instance=“127.0.0.1”,job="prometheus"}
http_requests_total{instance=“127.0.0.2”,job="prometheus"}
http_requests_total{instance=“127.0.0.2”,job="monitor"}
```
三种，那么结果rate(http_requests_total[5m])得到的结果也是三种，因为他们的标签不一样，属于3种时间序列，所以我们结果会有3种。如果我想得到这三个时间序列的每秒总请求数则可以

```
sum(rate(http_requests_total[5m]))
```

sum()是求和函数，可以将不同维度的时间序列聚合，得到的结果只有一种时间序列。现在我想将127.0.0.1和127.0.0.2这两个实例的请求数分别聚合既我想得到每台机器的每秒请求总数，可以使用promQL的分组功能

```
sum(rate(http_requests_total[5m])) by (instance)
```
这样我们得到的结果又两种时间序列。现在我想得到3中时间序列中每秒请求数最多的一个该怎么计算呢？如下

```
topk(1,http_requests_total[5m])
```

如果我想知道http_requests_total有多少种指标，可以

```
count(http_requests_total)
```
结果返回值为3.

## promQL高级

我们已经对promQL已经有了一定程度的了解，但是如果我们想使用的更加得心应手，则还需要对promQL有更加深入的了解。
### [数据类型](https://prometheus.io/docs/concepts/metric_types/)
在Prometheus中有3种数据类型

* **Instant vector** 即时向量，你可以看一个时间点的时序序列，它反映的是一个时间点不同标签的值组成的时间序列，如
 
```
http_requests_total
```

* **Range vector** 范围向量，你可以看做带有中括号时间限制的时序序列，它反映的是一段时间范围内的值组成的时间序列，时间单位可以是：s、m、h、d、w、y，如

```
rate(http_requests_total[5m])
```

* **Scalar** 标量，你可以把它看成一个float64位的数值

### 隐式标签
```
http_requests_total = {__name__ = 'http_requests_total'}
```
以上2个查询表达式是等价的

### 向量匹配
promQL中的数据类型是可以相互运算的即可以+,-,*,/……，它们的运算分为3种

* **标量 oprator 标量**
* **向量 oprator 标量**
* **向量 oprator 向量**

前面2种我们是很容易理解的，第3种它内部是如何计算的呢？接着基础篇的例子

```
http_requests_total - http_requests_total offset 5m
```
这个例子返回的是5分钟内请求的数量，返回的结果又几种呢？3种。因为它是根据标签来进行匹配的，即操作符两边的标签完全相同（不包括隐式标签，以__开头）的两个时间序列才进行运算。
