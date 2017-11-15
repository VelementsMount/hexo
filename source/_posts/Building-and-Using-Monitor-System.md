---
title: 监控系统的搭建和使用（二）
date: 2017/9/1 08:00:00
toc: true
tags: monitor
keywords: 
  - container
  - prometheus
  - monitor
categories: 容器监控
---

## 监控组件
一般性能监控系统会包含5大组件：
<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width=0 height=0 src="//music.163.com/outchain/player?type=2&id=1292916&auto=1&height=66"></iframe>

* **探针**：安装在应用中收集应用性能的包
* **收集器**：收集探针发送过来的数据或者主动拉取应用性能数据的工具
* **存储介质**：存储收集到的应用性能数据的介质
* **展示器**：将应用性能数据按照使用者的要求展示的工具
* **预警器**：当某一监测值超过预定阈值向devops成员发出预警

## 监控组件简要介绍

### prometheus
prometheus是一个开源的时序数据收集和处理的服务软件，它其实包含监控组件中的：收集器，存储和展示器，它所需的探针有相应的exporter和client library exporter组件提供，预警器可以由配套的alertmanager软件提供，它既可以监控各个主机的CPU、内存、文件等资源的使用情况，也可以监控服务的健康状况、网络流量，它既可以监控mysql、redis的使用状况，也可以定制监控信息，它是一个整体化解决方案的监控软件。
### cAdvisor
cAdvisor会收集、聚集、处理并导出运行中容器的信息,它可以为容器用户提供了了解运行时容器资源使用和性能特征的方法，安装后你可以通过web界面直接看到相应机器上容器中应用的资源使用和性能特征。它包含了监控组件中的前4大组件，但是它只能监控容器资源的使用和性能，因为它叫container advisor。
### grafana
grafana是一个开源的领先的能非常漂亮的展示时序数据分析结果的软件。它是使用js所写的一个软件，支持多种数据源，支持多种图形面板展示，它也提供了预警功能，是一个非常棒的图形展示组件。
### [exporter](https://prometheus.io/docs/instrumenting/exporters/)
下面介绍的是各种度量数据的导出器

* **node_exporter**: 监控一个主机的CPU、内存等资源的使用状况
* **[blackbox_exporter](https://github.com/prometheus/blackbox_exporter)**：监控各个服务的健康状况
* **mysqld_exporter**: 监控mysql的使用状况
* **redis_exporter**: 监控redis的使用状况
* **SNMP_exporter**: 监控网络流量

## 监控系统的搭建
我们需要建立一个全面的容器性能监控系统，所以我们选择的方案是『cAdvisor + Prometheus + Grafana』，下面我们将讲解整个监控系统的搭建。

* ***安装cAdvisor container***

通过docker-compose.yml的方式安装

```yaml
version: '2'
services:
  cadvisor:
    image: google/cadvisor
    restart: always
    ports:
      - 8080:8080
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
```

通过docker命令行的方式安装

```bash
docker run \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:rw \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --publish=8080:8080 \
  --detach=true \
  --name=cadvisor \
  --restart=always \
  google/cadvisor:latest
```
如果你在本机上安装的你可以通过`http://localhost:8080`访问cadvisor网页界面，你将看到![cAdvisor_Dashboard](http://ovqidxh71.bkt.clouddn.com/cAdvisor_Dashboard.png)


* ***安装prometheus***

```yaml
version: '2'
services:
  prometheus:
    image: prom/prometheus
    ports:
      - 9090:9090
    volumes:
      - /etc/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
```

prometheus.yml这个文件可以先配置成简单的监控prometheus自己，并把它放在/etc/prometheus/prometheus.yml这个路径。

```yaml
global:
  scrape_interval:     15s 
  evaluation_interval: 15s 
  external_labels:
      monitor: 'codelab-monitor'
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```
如果是在本机安装的你可以通过`http://localhost:9090`访问prometheus界面，你可以看到![prometheus_dashboard](http://ovqidxh71.bkt.clouddn.com/prometheus_dashboard.png)
你可以点击菜单栏中的Status下拉菜单中的Target标签看到监控的任务

* ***安装grafana***

```yaml
version: '2'
services:
  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
     - "3000:3000"
```
你可以通过`http://localhost:3000`访问grafana界面![grafana_dashboard](http://ovqidxh71.bkt.clouddn.com/grafana_dashboard.png)，输入默认用户名：admin密码：admin，可以进入管理界面

* ***安装node_exporter***

```bash
docker run -d -p 9100:9100 \
  -v "/proc:/host/proc:ro" \
  -v "/sys:/host/sys:ro" \
  -v "/:/rootfs:ro" \
  --net="host" \
  quay.io/prometheus/node-exporter \
    --collector.procfs /host/proc \
    --collector.sysfs /host/sys \
    --collector.filesystem.ignored-mount-points "^/(sys|proc|dev|host|etc)($|/)"
```

* ***安装blackbox-exporter***

```
docker run -d \
	 --restart=always \
	 -p 9115:9115 \
	 --name blackbox_exporter 
	 prom/blackbox-exporter:latest
```

其他的exporter可以根据自己的需求进行安装，其中cAdvisor和node-exporter需要在你需要监控的每个机器上安装，prometheus、blackbox-exporter、grafana部署一份就可以了。经过以上的安装部署，我们已经把容器性能监控的轮廓搭建好了，但是我们还没有将它们连接起来，下面我们将讲解它们的连接和使用

## 监控系统的连接和使用
我们首先需要将各种数据导出器、cAdvisor中的数据导入prometheus，因为prometheus采用的是主动拉取目标主机的监控数据，所以只需要让prometheus知道被监控的目标主机在哪就行，我们只需将Prometheus的配置文件改成下面这样：

```yaml
global:
  scrape_interval:     15s 
  evaluation_interval: 15s 
  external_labels:
      monitor: 'codelab-monitor'
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['192.168.77.153:9100']
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['192.168.77.153:8080']
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]  # Look for a HTTP 200 response.
    static_configs:
      - targets:
        - https://www.baidu.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 192.168.77.153:9115  # Blackbox exporter

```
***注意***：192.168.77.153 是本机的局域网地址，因为我们都是通过容器启动，所以不要使用localhost。

重启后，你可以点击菜单栏中的Status下拉菜单中的Target标签看到监控的任务，确认是否启动成功。接下来我们需要将prometheus里面的数据导入grafana，grafana可以作为一个所有数据集中展示的地方，因为很可能未来我们的数据来源不一定来自一个prometheus甚至可能是其他的存储库

我们先登录到grafana管理界面，创建第一个数据源，![datasource_dashboard](http://ovqidxh71.bkt.clouddn.com/datasource_dashboard.png)数据源创建好了以后我们需要创建dashboard面板，我们直接去[grafana](https://grafana.com/dashboards?search=cAdvisor)官网搜索cAdvisor的dashboard和node exporter的dashboard。![cadvisor](http://ovqidxh71.bkt.clouddn.com/cadvisor.png)
 ![nodeexporter](http://ovqidxh71.bkt.clouddn.com/nodeexporter.png)
并将它们分别导入grafana。![dashboard](http://ovqidxh71.bkt.clouddn.com/dashboard.png)

导入成功后我们将会看到监控的数据![nodeExpoerter](http://ovqidxh71.bkt.clouddn.com/nodeExpoerter.png)![cadvisor_dash](http://ovqidxh71.bkt.clouddn.com/cadvisor_dash.png)

在garafana的官网面板中我们没有找到black-exporter相关的面板，所以我们需要书写一些查询语句将我们监控www.baidu.com这个网址的健康状况的信息反映出来，我们可能会为官网贡献一套black-exporter模板。

到现在为止我们已经将整个容器监控系统搭建起来，我们的监控系统具有监控各个服务器CPU、内存、文件等资源使用状况，服务器中每个容器的CPU、内存、文件等资源使用状况以及对我们开发的应用进行健康检查的能力。我们做的非常的完美，是的，我们开发的应用终于可以上线了。这一切看似完美的背后仍然存在一点小小的缺憾，比如我想知道各个微服务的被调用情况，每次请求的延迟，每次请求正确与否，一段时间内的请求量……；仔细观察，我们还会发现一个问题，就是在prometheus的配置文件中我们监控的地址都是固定的，我们知道在docker容器集群中，这些地址是随时变动的，还有我们增加一个服务就需要去更改prometheus的配置文件，that is too bad！我需要prometheus可以动态更改监控的地址，动态增加监控的服务，I need more！
