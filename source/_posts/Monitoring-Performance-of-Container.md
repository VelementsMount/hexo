---
title: 容器监控(一)
toc: true
date: 2017/9/1 08:00:00
tags: monitor
keywords: 
  - container
  - prometheus
  - monitor
categories: 容器监控
---

> **“You can’t manage what you don’t measure"**
> 
>**— W. Edwards Deming**

近年来，以[docker](https://www.docker.com/)为首的容器技术在IT领域尤其是在云计算和微服务应用领域掀起了一股狂潮，成为当下特别流行的一种技术，说明容器技术正好满足了当今软件领域中的一些迫切需求，它主要解决了下面的一些问题

* 它将应用及依赖的运行环境打包成镜像，消除了线上线下环境的差异，保证了应用环境的一致性；
* 作为一种轻量级的虚拟化技术，它以很小的代价却提供了不错的资源隔离和限制能力，可以更加细粒度的分配资源，极大提高了资源的使用率；
* 还有它构建一次，到处运行，提高了容器的跨平台性的同时，大大简化了持续集成、测试和发布的过程……


这些优势已经给当今软件的开发、测试、部署、运维带来了一场变革，同时也极大简化了软件的获取、学习、使用及交流。这种容器化技术带来极大便利性的同时，也给传统的运维带了冲击，尤其是对容器化集群的监控，更是给传统的运维带来了不小的挑战。

正如开篇提到的：你无法管理你不能测量的事物，这句话道出了应用监控的精髓。在devops的世界里，禁止上线没有受到监控的应用，因为你不知道它会做什么以及别人会对它做什么，那么对于大量使用容器技术的我们如何对容器进行监控就成了我们必须要解决的问题。

## 容器监控的解决方案
在**容器时代**，传统的那些耳熟能详的监控软件如[Zabbix](https://www.zabbix.com/)不能提供方便的容器化服务的监控体验，相反的许多新生的开源监控项目则将对容器监控的支持放到了关键特性的位置，如[InfluxDB](https://www.influxdata.com/)，[Prometheus](https://prometheus.io/)等获得了广泛的认可。

在DockerCon EU 2015上，Swisscom AG的云方案架构师Brian Christner阐述了“Docker监控”的概况，分享了这方面的最佳实践和Docker stats API的指南，并对比了三个流行的监控方案：[cAdvisor](https://github.com/google/cadvisor)、“cAdvisor + InfluxDB + [Grafana](https://grafana.com/)”以及Prometheus。其中Prometheus是整体化的开源监控软件，但它本身对容器信息的收集能力以及图表展示能力相比其他专用开源组件较弱，通常在实际实施的时候依然会将它组合为『cAdvisor + Prometheus』或**『cAdvisor + Prometheus + Grafana』**的方式使用。

prometheus作为天生的容器监控的项目，特别适合作为度量数据的存储和查询，所以我们选择了以prometheus为核心的容器监控系统的解决方案。

我们后端大量采用java和nodejs两种开发语言，java后端采用[Spring Cloud](http://projects.spring.io/spring-cloud/)支撑的一套微服务架构，我们需要将开源的监控软件与本公司的实际技术相结合。坐而言不如起而行，接下来我们会分三篇文章**监控系统的搭建和使用**、**prometheus基础**和**自定义监控数据**详细介绍『cAdvisor + Prometheus + Grafana』这种解决方案在本公司的本土化过程。

















