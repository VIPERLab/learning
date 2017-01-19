# 目录
* [前言](#前言)
* [总体结构](#总体结构)
* [filebeat](#filebeat)
	* [filebeat.yml](#filebeat.yml)
	* [filebeat启动方式](#filebeat启动方式)
* [logstash](#logstash)
	* [logstash.yml](#logstash.yml)
	* [logstash.conf](#logstash.conf)
	* [logstash启动方式](#logstash启动方式)
* [elasticsearch](#elasticsearch)
* [kibana](#kibana)
	* [kibana.yml](#kibana.yml)
	* [kibana启动方式](#kibana启动方式)
	* [kibana页面使用](#kibana页面使用)
		* [Configure an index pattern](#configure-an-index-pattern)
		* [Discover](#discover)
		* [Visualize](#visualize)
		* [Dashboard](#dashboard)
* [告警脚本](#告警脚本)

# 前言
春哥开源的[OpenResty](http://openresty.org/cn/)使得nginx不仅用于代理，还可以快速开发业务server。网上大多数对nginx进行系统或业务层面的监控都是付费的。Elastic Stack是目前流行的开源的日志收集组件，本文探索了使用Elastic Stack搭建nginx的access log日志平台和基于日志实现业务层面的监控。下面几张日志平台和告警信息的效果图：

* 访问量统计

![访问量](https://github.com/cantoo/learning/raw/master/elasticsearch/access.png)

* 请求耗时统计

![请求耗时](https://github.com/cantoo/learning/raw/master/elasticsearch/requesttime.png)

* 分析客户端IP所在地域的热力图

![热力图](https://github.com/cantoo/learning/raw/master/elasticsearch/heatmap.png)

* 告警信息

![告警信息](https://github.com/cantoo/learning/raw/master/elasticsearch/alarm.png)


# 总体结构

![总体结构](https://github.com/cantoo/learning/raw/master/elasticsearch/structure.png)

- **filebeat**: 读取日志文件上报到logstash。
- **logstash**: 分析日志，使用正规匹配提取拆解日志中的字段，去除一些不必要的字段，丰富一些字段，转换一些字段的类型。
- **elasticsearh**: 存储分析后的日志，供外部查询。
- **kibana**: 图表展示日志统计信息。
- **告警脚本**：查询异常日志，发送告警信息。

**如果日志量大，可以在filebeat和logstash中间加redis缓存，达到使logstash负载平滑的目的。**


# filebeat

## filebeat.yml

```yml
filebeat:
  prospectors:
	-
	paths:
		- "/data/app/nconn/logs/nconn_svrd.log"
	input_type: log
	encoding: "utf-8"
	document_type: campusx

	exclude_lines: [' \[info\] ', ' \[debug\] ', ' \[error\] ', ' \[warning\] ', ' +0800 ']

	#multiline:
	   #pattern: ^201
	   #negate: true
	   #match: after

output:
	logstash:
		hosts: ["10.223.xxx.xxx:5077"]

logging:
	level: warning
	# enable file rotation with default configuration
	to_files: true
	# do not log to syslog
	to_syslog: false

	files:
		path: /data/logs/filebeat
		name: campusx.log
		keepfiles: 2
		rotateeverybytes: 10485760 # = 10MB
```

- **prospectors.paths**：定义日志文件的位置或路径。
- **prospectors.input_type**：log按行读取日志文件。
- **prospectors.document_type**：输出到elasticsearch数据的type字段的值，与logstash output配置中模板文件中类型的名称一致。
- **prospectors.exclude_lines**：去除匹配指定表达式的行（这里我们只关注nginx的access log，故把匹配error log的行忽略）。
- **prospectors.multiline**：如果一条日志在日志文件中存在有多行的情况，这里定义如何标识一条日志。
- **output.logstash**：定义输出到logstash的配置。
- **logging**：filebeat运行日志的配置。

[官方filebeat配置文件说明](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-configuration-details.html)

## filebeat启动方式

```bash
cd /your/root/to/filebeat
nohup ./filebeat -d ERROR -c filebeat.yml & 
```

# logstash

logstash相当于是数据管道，从filebeat接收数据，处理后，输出给elasticsearch。

- **logstash.yml**：logstash的运行配置。
- **logstash.conf**：logstash的管道配置。
- **logstash.template.json**：输出到elasticseach数据的索引配置。

## logstash.yml

```yml
pipeline.workers: 4

pipeline.output.workers: 4
```

[官方logstash运行配置详情说明文档](https://www.elastic.co/guide/en/logstash/current/logstash-settings-file.html)

[官方logstash性能调优文档](https://www.elastic.co/guide/en/logstash/current/performance-tuning.html)

## logstash.conf

```conf
input {
    beats {
        port => 5077
    }
}

filter {
    grok {
        patterns_dir => ["/your/root/to/logstash/config/pattern_dir"]
        match => ["message", "%{NGINXACCESS}"]
    }
 
    if "_grokparsefailure" in [tags] {
        drop {}
    }
   
    date {
        match => ["ngx_localtime", "yy-MM-dd HH:mm:ss"]
        target => "@timestamp"
    }

    geoip {
        source => "remote_addr"
        fields => [ "ip", "country_name", "region_name", "city_name", "location" ]
    }

    # remove unnecessary fields
    mutate {
        remove_tag => [ "_grokparsefailure", "beats_input_codec_plain_applied"]
        remove_field => ["@version", "beat", "source", "tags", "input_type", "message", "host", "offset", "count", "%{somefield}.keyword"， "ngx_localtime", "remote_addr", "msec", "httpversion", "request_body", "http_referer", "http_x_forwarded_for", "upstream_addr"]
        convert => { "msec" => "float" "status" => "integer" "body_bytes_sent" => "integer" "request_length" => "integer" "request_time" => "float" "upstream_response_time" => "float" }
        split => { "x_client" => "," }
    }
} 

output {
    elasticsearch {
        hosts => ["10.223.xxx.xxx:9201"]
        template => "/your/root/to/logstash/config/filebeat.template.json"
    }

    #stdout {
    #    codec => rubydebug {
    #        metadata => false
    #    }
    #}
}
```

logstash是管道，主要是输入，过滤，输出三块配置。

- **input**
	- **beats.port**: 接收filebeat上报数据的TCP端口。

	[input插件官方文档](https://www.elastic.co/guide/en/logstash/current/input-plugins.html)

- **filter**
	- **grok**：尝试用一个正则表达式匹配日志，提取日志中的字段。
		grok使用自己的一套正则表达式语法，官方github开源了一些常用的正规表达式，请参见[这里](https://github.com/logstash-plugins/logstash-patterns-core/tree/master/patterns)。
		[grok debugger](https://grokdebug.herokuapp.com)是一个在线调试grok正则表达式的工具。
		
		如nginx的log format为：
		```conf
		log_format main '$ngx_localtime "$server_addr:$server_port" "$remote_addr" $msec "$request" "$request_body" '
		       '$status $body_bytes_sent $request_length "$http_referer" '
		       '"$http_user_agent" "$http_x_forwarded_for" $request_time '
		       '$upstream_response_time "$upstream_addr" "$http_x_user" "$http_x_client" "$mylog"';
		```
		可以用以下的正则表达式匹配。
		```bash
		# /your/root/to/logstash/config/pattern_dir
		NGINXACCESS %{DATESTAMP:ngx_localtime} "%{IP:server_addr}:%{POSINT}" "%{IP:remote_addr}" %{NUMBER:msec} "(?:%{WORD:verb} %{URIPATH:path}(?:%{URIPARAM:args})?(?: HTTP/%{NUMBER:httpversion})?|-)" %{QS:request_body} %{NUMBER:status} %{NUMBER:body_bytes_sent} %{NUMBER:request_length} %{QS:http_referer} "%{DATA:http_user_agent}" %{QS:http_x_forwarded_for} %{NUMBER:request_time} (?:%{NUMBER:upstream_response_time}|-) %{QS:upstream_addr} "%{DATA:x_user}" "%{DATA:x_client}"
		```

	- **if**：这里我们把grok匹配失败的日志全部丢弃。\
	- **date**：logstash是按天划分日志的索引的，所以分析的字段中一定要有一个时间字段，这个日期字段通过date转换成logstash划分索引使用的@timestamp字段。
	- **geoip**：geoip插件可以把IP地址转换为地域信息，以统计访问来源的地域分布，source指定使用哪个字段转换，fields指定转换后保留哪些字段。geoip默认使用logstash的IP库，也可以使用自定义的IP库，设置方法见[官方文档](https://www.elastic.co/guide/en/logstash/current/plugins-filters-geoip.html)。
	- **mutate**：使用mutate移除一些不必要的数据标签，不感兴趣的字段，按某个分隔符分隔某个字段的值，让该字段成为一个多值字段。mutate还可以转换一些字段的数据类型，这些类型和后面output插件中的模板文件中类型的描述要一致。

	[filter插件官方文档](https://www.elastic.co/guide/en/logstash/current/filter-plugins.html)

- **output** 
	- **elasticsearch**：输出到elasticsearch，指定elasticsearch的地址，和索引使用的模板文件。如上面filter插件处理完的数据的索引模板为：

	```json
	{
	    "template": "logstash-*",
	    "settings": {
	        "number_of_replicas": 0
	    },
	    "mappings": {
	        "campusx": {
	            "_all": {
	                "enabled": false
	            },
	            "dynamic": false,
	            "properties": {
	                "@timestamp": {
	                    "type": "date"
	                },
	                "x_user": {
	                    "type": "keyword"
	                },
	                "body_bytes_sent": {
	                    "type": "integer"
	                },
	                "http_user_agent": {
	                    "type": "keyword"
	                },
	                "path": {
	                    "type": "keyword"
	                },
	                "args": {
	                    "type": "keyword"
	                },
	                "request_time": {
	                    "type": "scaled_float",
	                    "scaling_factor": 1000
	                },
	                "verb": {
	                    "type": "keyword"
	                },
	                "server_addr": {
	                    "type": "keyword"
	                },
	                "request_length": {
	                    "type": "integer"
	                },
	                "upstream_response_time": {
	                    "type": "scaled_float",
	                    "scaling_factor": 1000
	                },
	                "x_client": {
	                    "type": "keyword"
	                },
	                "status": {
	                    "type": "integer"
	                },
	                "geoip": {
	                    "properties": {
	                        "ip": {
	                            "type": "keyword"
	                        },
	                        "country_name": {
	                            "type": "keyword"
	                        },
	                        "region_name": {
	                            "type": "keyword"
	                        },
	                        "city_name": {
	                            "type": "keyword"
	                        },
	                        "location": {
	                            "type": "geo_point"
	                        }
	                    }
	                }
	            }
	        }
	    }
	}

	```

	**settings.number_of_replicas = 0**，副本数量，指定0节省存储空间。
	**mapping.campusx._all.enable = false**，禁用全文索引，日志查询不需要全文索引。
	**mapping.campusx.dynamic = false**，禁用动态创建字段。

	[output插件官方文档](https://www.elastic.co/guide/en/logstash/current/output-plugins.html)

## logstash启动方式

```bash
nohup bin/logstash -f config/logstash.conf &
```


# elasticsearch
为优化elasticsearch性能，需要调整一些系统，java，运行配置，请参考[elasticsearch用法简述](https://github.com/cantoo/learning/blob/master/elasticsearch/elasticsearch%E7%94%A8%E6%B3%95%E7%AE%80%E8%BF%B0.md)。


# kibana

## kibana.yml

```yml
# Kibana is served by a back end server. This setting specifies the port to use.
server.port: 5602

# Specifies the address to which the Kibana server will bind. IP addresses and host names are both valid values.
# The default is 'localhost', which usually means remote machines will not be able to connect.
# To allow connections from remote users, set this parameter to a non-loopback address.
server.host: "10.223.xxx.xxx"

# The URL of the Elasticsearch instance to use for all your queries.
elasticsearch.url: "http://10.223.xxx.xxx:9201"
```

## kibana启动方式

```bash
cd /your/root/to/kibana
nohup bin/kibana & 
```

## kibana页面使用

### Configure an index pattern
首次打开kibana，需要配置一个索引的结构。在`Management->Index Patterns`选择`Time-field name`为`@timestamp`后，点击create即可。可以看到kibana已经自动拉取了logstash output配置中模板的字段列表。

![Configure an index pattern](https://github.com/cantoo/learning/raw/master/elasticsearch/config_an_index_pattern.png)

![Logstash index pattern](https://github.com/cantoo/learning/raw/master/elasticsearch/logstash_index_pattern.png)

### Discover

配置好index pattern后，即可对Discover页对日志进行搜索。下面是几种常用的搜索：

* 所有返回500以上的access log：`status:[500 TO 600]`
* 搜索所有`/api/v1/users`接口的访问：`path:/\/api\/v1\/users.*/`
* 多个条件的AND搜索：`status:[500 TO 600] AND path:/\/api\/v1\/users.*/`

![Discover](https://github.com/cantoo/learning/raw/master/elasticsearch/discover.png)

### Visualize

Visualize用于展示日志的统计图表，如使用timeline panel把当前，一天前和七天前的访问量展示[前言-访问量统计](#前言)所示的图片中。

```timelion
.es().label("Current"),.es(offset=-1d).label("1 day before"),.es(offset=-7d).label("7 day before")
```

### Dashboard
使用Visualize创建了多个图表后，可以集中添加到Dashboard中一起展示，如把访问量统计和耗时统计一起展示：

![Dashboard](https://github.com/cantoo/learning/raw/master/elasticsearch/dashboard.png)

可以在Dashboard的搜索框中输入搜索条件，点击搜索后Dashboard中所有图表的数据都切换成指定搜索条件的。如我们搜索`	`即可得到`/api/v1/users`接口的访问量统计和耗时统计。

# 告警脚本
告警脚本使用elasticsearch的Query DSL从elasticsearch查询异常数据，然后通知相关责任人。
elasticsearch Query DSL的使用可以参考[elasticsearch用法简述](https://github.com/cantoo/learning/blob/master/elasticsearch/elasticsearch%E7%94%A8%E6%B3%95%E7%AE%80%E8%BF%B0.md)或[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html)。
如下面的告警脚本，查询近1小时内返回500以上的次数，如果超过100，就发出告警信息：

```lua
-- /data/app/nginx/bin/resty campusx_alarm.lua
local cjson = require("cjson.safe")

local now = ngx.time()
local index = os.date("%Y.%m.%d", now - 8*3600)

local rsp, err = http_request("http://10.223.xx.xx:9201/logstash-" .. index .. "/campusx/_search", {
		query = {
			bool = {
				must = {
					{
						range = {
							status = {
								gte = 500,
								lt = 600
							}
						}
					},
					{
						range = {
							["@timestamp"] = {
								gte = "now-1h/m",
								lt = "now/m"
							}
						}
					}
				}
			}
		},
		from = 0,
		size = 1000,
		_source = { "path" }
	}):execute()

if err then
	ngx.log(ngx.ERR, "search failed")
	return
end

rsp = cjson.decode(rsp)
if not rsp or type(rsp) ~= "table" or type(rsp.hits) ~= "table" or type(rsp.hits.total) ~= "number" then
	ngx.log(ngx.ERR, "rsp decode failed")
	return
end

ngx.log(ngx.INFO, "now=", now, ",over 500=", rsp.hits.total)
if rsp.hits.total > 100 then
	-- 发送告警信息
end

```





