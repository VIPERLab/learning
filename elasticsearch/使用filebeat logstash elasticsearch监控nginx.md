# 前言
春哥开源的[OpenResty](http://openresty.org/cn/)使得nginx不仅用于代理，还可以快速开发业务server。网上大多数对nginx进行系统或业务层面的监控都是付费的。Elastic Stack是目前流行的开源的日志收集组件，本文探索了使用Elastic Stack搭建nginx的access log日志平台和基于日志实现业务层面的监控。下面是同学我来了APP后台日志平台和告警信息的效果图：

* 访问量统计
![访问量](https://github.com/cantoo/learning/raw/master/elasticsearch/access.png)

* 请求耗时统计
![请求耗时](https://github.com/cantoo/learning/raw/master/elasticsearch/requesttime.png)

* 分析客户端IP所在地域的热力图
![热力图](https://github.com/cantoo/learning/raw/master/elasticsearch/heatmap.png)

* 告警信息
![告警信息](https://github.com/cantoo/learning/raw/master/elasticsearch/alarm.png)

# 总体结构

[nginx -> log <- filebeat] -> logstash -> elasticsearch <- kibana, 告警脚本


