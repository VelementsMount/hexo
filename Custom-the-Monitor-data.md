---
title: 自定义监控数据（四）
date: 2017/9/1 08:00:00
toc: true
tags: monitor
keywords: 
  - container
  - prometheus
  - monitor
categories: 容器监控
---

<h4 align='center'>作者：萧超杰 刘志林 朱枫翔</h4>

## 自定义监控数据
需求是永无止境的，我们一定要在正确的时间做正确的事情。prometheus作为容器监控系统，它的功能是强大的，我们可以通过它提供的客户端开发包，自定义我们需要监控的性能数据；我们也可以通过它提供的[file_sd_configs](https://prometheus.io/docs/operating/configuration/#%3Cfile_sd_config%3E)动态配置监控的地址，动态增加监控的服务，这正好解决了前面章节我们提出的2个明确的需求。下面就让我们开启这一场神奇的解决问题和coding之旅吧。


### 自定义监控数据的目标
我们自定义监控数据的目标是在15秒内发现系统的稳定性问题并主动的去解决问题。其实就是在最短的时间内发现由于应用发布、应用内部bug、应用崩溃自重启等等带来的系统的稳定性问题。最终我们搭建好的监控系统在实际的生产环境中是15s去拉取一次度量指标，这个时间最短可以调到1s一次，所以说我们的系统是一个准实时或者说是一个实时的监控系统，它带来解决问题的一个巨大的革新。


### prometheus+grafana+zuul的监控
我们后端的微服务采用的是springcloud支撑的微服务架构，我们的网关为zuul，所以任何请求都会经过zuul来做转发，我们监控的思路是，在zuul网关层增加一个过滤器，记录每个请求的模块，每个请求的路径，每个请求的方法，每个请求的响应时间，每个请求的次数。然后将这些数据导入到prometheus，grafana再将prometheus中的数据以图表的形式展示出来，如下图![Prometheus-Grafana.png](http://ovqidxh71.bkt.clouddn.com/Prometheus-Grafana.png)。这个问题的难点在于怎样将zuul中的度量数据导出和怎样在grafana中以合适的方式展现。下面我们就来探讨这2个问题的解决方法。

#### 导出zuul中的度量数据
我们使用的是spring框架，spring框架中已经有了一个spring-acurator模块，引入这个模块，我们就可以在/metrics的路径下得到json格式的度量数据![Metrics-Json.png](http://ovqidxh71.bkt.clouddn.com/Metrics-Json.png)，我们可以使用spring-acurator自带的CounterService和GuageService将我们需要记录的数据记录到/metrics
路径下。但是我们知道prometheus不接受json格式的数据，它接受的是纯文本格式的数据,例如![Prometheus-metrics.png](http://ovqidxh71.bkt.clouddn.com/Prometheus-metrics.png)，如何将json格式的数据转换为prometheus接受的纯文本格式我们可以参考Johan Zietsman写的[Actuator and Prometheus](http://blog.monkey.codes/actuator-and-prometheus/),我们自定义了spring-acurator一个新的endpoint：/prometheus,在这个页面我们导出了prometheus可以接受的度量数据，似乎一切都很美好，但是我们深入的应用之后发现，spring-acurator提供的CounterService和GuageService实在是太简陋了，它完全无法和prometheus提供的客户端的包相比

```
compile('io.prometheus:simpleclient:0.0.26')
compile('io.prometheus:simpleclient_common:0.0.26')
```
spring-acurator中的CounterService记录counter类型每次只能增加1，完全成了一个计数器，如果我想统计所有请求的总时间，它就无法胜任,还有Histogram，Summary类型的数据都有一定残缺。所以最后我们选择是使用prometheus的客户端的包做数据统计，扩展spring-acurator的抽象接口AbstractEndpoint和AbstractEndpointMvcAdapter将这些度量数据暴露成一个Endpoint。最终我们达到了我们想要的结果。

在我们导出的prometheus数据格式主要有3种

```
http_response_time_milliseconds_count{method="ytx_sso_users",module="ytx_sso",status="200",method_type="GET",} 166.0  
http_response_time_milliseconds_sum{method="ytx_sso_users",module="ytx_sso",status="200",method_type="GET",} 110186.0  
http_total_request_size{status="200",module="cms_content",} 2162.0

```
`http_response_time_milliseconds_count`统计每个接口的调用次数
`http_response_time_milliseconds_sum`统计每个接口响应时间的总和
`http_total_request_size`统计每个模块的调用总次数
以上三种都是Counter类型，前两种通过Summarry统计的，最后一种使用Counter统计的。
labels标签：method记录访问的路径，module记录访问的模块，status记录响应的状态码，method_type记录请求的类型。

#### 在grafana中展示数据
grafana的使用可以到[grafana官网](http://docs.grafana.org/)学习，有时间我们会详细介绍。
我们仅仅讨论怎样书写promQL表达式将我们的数据按照我们要求完美的展现出来，比如我想统计1分钟内每个服务的可用性。promQL如下：

```
(1+sum(http_total_request_size{status="200"}) by (module)-sum(http_total_request_size{status="200"} offset 30s) by (module)) * 100/ (1+sum(http_total_request_size{status=~"200|500"})by (module) -sum(http_total_request_size{status=~"200|500"} offset 30s)by (module) )
```
每个服务的吞吐率：

```
sum(rate(http_response_time_milliseconds_count{kubernetes_namespace="$namespace"}[1m])) by (module) 
```
还有错误调用Top10

```
topk(10,sum(http_response_time_milliseconds_count{status!="200"}) by(method,method_type,status)  - sum(http_response_time_milliseconds_count{status!="200"} offset 1m) by(method,method_type,status))
```
等等，在这里就不一一列举。最终我们得到的结果为![result_show1.png](http://ovqidxh71.bkt.clouddn.com/result_show1.png) ![result_show2.png](http://ovqidxh71.bkt.clouddn.com/result_show2.png) ![result_show3.png](http://ovqidxh71.bkt.clouddn.com/result_show3.png)

#### 优势
现在一个实时的监控系统已经搭建起来，它带来的优势显而易见，我们可以实时观测各个服务的健康状况，一目了然。同时它带来了解决问题方式的改变，以往没有实时监控的时候，我们的服务某个接口报错一定需要等到用户反馈或者调用方反馈，我们才能发现并被动的解决问题。现在我们可以清晰的看到那个接口调用报错，在用户或者调用方还没有反馈之前就已经发现了问题并主动的解决掉问题。这带来解决问题方式的一个革新。

### prometheus+eureka的集成
Prometheus因为是监控容器的性能软件，它已经集成了很多主流的常见的服务集群服务的动态发现机制，比如：k8s，consul等的注册发现，但是没有包含springcloud组件中eureka的注册发现机制。我们知道，作为docker容器化部署，尤其在容器化集群中，服务的地址不是固定的，所以我们必须要解决动态获取eureka中zuul网关的地址。我们可以通过eureka主机上的路径/eureka/apps/ZUUL-SERVER拉取到多个zuul网关的实例的ip地址，并将其配置到prometheus中。怎样打通这样一个链路呢？我们发现了prometheus中有一个file_sd_configs的标签,可以配置一个文件，prometheus可以动态去读取这个文件，获取里面的目标地址。所以打通这个链路的原理图![](http://ovqidxh71.bkt.clouddn.com/dynamic_config_2.png)

找到了解决问题的方法，那么我们就开始实践。
#### 将zuul网关的地址打印到文件中
我们使用node.js写了一个脚本，每隔10s去拉去一下eureka上zuul网关的ip地址，将其答应到config.json文件中。nodejs的代码就不在此展示了，最终打印到config.json中的内容

```
[{"targets":["192.168.77.153:8041","192.168.77.164:8041","192.168.77.156:8041"]}]
```
启动这个脚本程序
#### 配置将prometheus动态读取文件内容

prometheus中的配置文件内容为

```
file_sd_configs:    #file dynamic discovery
  - files:
    - /config/config.json
  	refresh_interval: 30s
```
修改配置文件以后，重新启动，在Status菜单下的targets标签下可以观察到prometheus动态拉取到了zuul的地址，大大减小了运维的成本。


## prometheus查询性能的优化

当我们以为一切都万事大吉的时候，在使用过程中，我们发现了一个有点严重的问题，就是grafana的dashboard监控面板需要很长一段时间才能将我们需要的监测结果给显示出来，
这是非常致命的，因为我们这个prometheus是一个实时监控平台，如果需要花费3-5分钟才将数据显示出来就失去了实时性的意义了，所以对我们来说这是一个必须要解决的问题。下面我们将整个解决问题的过程呈现出来供大家参考。

prometheus使用的是本地磁盘存储，它非常耗内存和cpu，生产上的数据是相当庞大的，当我们写了一个性能非常不好promQL时，prometheus将进行大量的计算，虽然prometheus已经针对时序数据的计算做了优化，但是仍然阻止不了我们写出性能很差的查询语句。

### 第一阶段优化
我们一开始给指标名起名的格式为：指标功能_模块名_访问方法_路径，没有打label，最后数据展示的时候我们大量使用{__name__ =~ ".\*api.\*get_.\*"}去正则过滤，最后导致的结果是![First_CPU_Load](http://ovqidxh71.bkt.clouddn.com/First_CPU_Load.png)我们可以看到prometheus那台机器的cpu负载降不下来，grafana刷新一次就发一次promQL查询请求到prometheus去计算，当结果有没有出来，我们又刷新页面，又发一次查询请求，这样形成了一个恶性循环。最后我们的改进措施是，将模块名、请求方法、返回状态吗、请求类型都设置为标签，这正如你上面看到的，指标名仅仅指代测量的指标。

### 第二阶段的优化
上面的优化措施过后，我们的查询性能大大提高，在开发和测试环境中已经非常流畅的能显示出图片，但是当我们的机器到了生产上面之后，我们发现生产的数据量很巨大，grafana的显示速度还算能接受，但是prometheus那台机器的cpu负载间歇性突增![Second_CPU_Load](http://ovqidxh71.bkt.clouddn.com/Second_CPU_Load.png)我们发现可能是我们生产上是抓取的时间间隔缩短，同时抓取的数据量非常庞大，每一次发送请求都会进行大量的计算，导致cpu负载间歇性突增。为了改善性能，我们第一步将大量我们没有使用到的自定义监测数据删掉不予记录，prometheus抓取的数据大大减少；第二步我们将可以在客户端聚合的数据先在客户端聚合好，然后在存入prometheus，而不是每一次请求都让prometheus去计算聚合一次，比如统计每个模块的返回值200和500状态码的调用总次数  
**优化后**

```  
sum(http_total_request_size{status=~"200|500"})by (module) 
```
**优化前**
```
sum(http_response_time_milliseconds_count{status="200|500"}) by (module)
```

结果![Last_CPU_Load](http://ovqidxh71.bkt.clouddn.com/Last_CPU_Load.png)
真的非常完美。


##  写在最后
经过以上的搭建过程和开发过程，我们已经非常完美的搭建了一个实时的容器性能监控系统，能够很好的完成我们对生产上应用的监控。在本篇文章中还有很多方面没有详述，如prometheus的配置、grafana的使用、预警的设置、grafana变量模板的使用……，大家可以自己去研究探索。我们是银天下devops团队，如果你有任何疑问和技术问题，欢迎与我们联系！
