---
title: prometheus监控spark on yarn方案
date: 2018-06-22 08:35:32
tags: spark
categories: 技术
---
# 监控指标
    
使用sparkMetricSink监控的指标

# 考虑问题
    
- spark自带的sink使用io.dropwizard.metrics，目前不支持prometheus
- spark自带的metrics名称格式为：appId.instance.[appName].XX，一旦应用重启，指标就会重置
- spark on yarn目前没有好的办法支持prometheus的动态发现

<!-- more -->
# 开发过程
 ## 1. 开发PrometheusMetricsServlet

(1).此类继承spark目前已有的MetricsServlet，MetricsServlet为spark默认开启的sink通过/metrics/json可访问json格式的metrics；并且spark只能开启一个servletsink，所以必须配置中必须指定：*.sink.servlet.class=org.apache.spark.metrics.sink.PrometheusMetricsServlet

(2). 引入prometheus dropwizard client
```
        <dependency>
            <groupId>io.prometheus</groupId>
            <artifactId>simpleclient_dropwizard</artifactId>
         </dependency>
```
(3). 在getMetricsSnapshot方法内，将MetricRegistry转换成prometheus的结构，在转换中去掉metrics中key里面的appid，write004方法仿照TextFormat改造；
```
override def getMetricsSnapshot(request: HttpServletRequest): String = {
  val stringWriter = new StringWriter();
  PrometheusMetrics.write004(stringWriter,collectorRegistry.metricFamilySamples())
  stringWriter.close()
  stringWriter.getBuffer.toString
}
```
(4). 以上修改完成后，启动应用访问yarn_url/proxy/application_XXX_X/metrics/json页面，即可以看到prometheus结构数据

 ## 2. 读取yarn信息生成json文件
使用python从yarn读取spark的地址信息，并加上appName标签，按照队列名称生成文件，代码如下：
```
# coding:utf-8
import json
from httplib import HTTPConnection

urls = ["rm1.com", "rm2.com"]
queues = ["queue"]


# 解析json获取applications信息
def applications_info(path, method="GET"):
    for url in urls:
        try:
            conn = HTTPConnection(host=url, port=8088, timeout=3)
            conn.request(method, path)
            response = conn.getresponse()
            jsonobj = json.loads(response.read())
            conn.close()
            if jsonobj:
                return jsonobj["apps"]["app"]
        except Exception, e:
            # print e
            pass


if __name__ == "__main__":
    for queue in queues:
        apps = applications_info(
            "/ws/v1/cluster/apps?states=RUNNING&type=spark&queue={0}".format(queue))
        result_list = []
        if apps:
            for app in apps:
                appid = app["id"]
                appname = app["name"].replace("-", "_")
                trackingUrl = app["trackingUrl"][7:]
                index = trackingUrl.find("/")
                result_list.append({"targets": [trackingUrl[0:index]],
                                    "labels": {"appName": appname,
                                               "__metrics_path__": "{0}metrics/json".format(trackingUrl[index:])}
                                    })
        print(result_list)
        fo = open("spark/{0}.json".format(queue), "w")
        fo.write(json.dumps(result_list, indent=4))
        fo.close()

```
生成的json格式如下，因为prometheus的target只能包含ip和port，通过"\_\_metrics_path__" 指定具体访问路径：
```
[
    {
        "labels": {
            "__metrics_path__":"/proxy/application_XXX_X/metrics/json",
            "appName":"app_name"
        },
        "targets": [
            "yarn_url:8088"
        ]
    }
}
```
 ## 3. 配置prometheus动态文件
prometheus支持多种动态发现配置，此处使用file_sd_configs文件动态发现
```
file_sd_configs:
      - files: ['spark/*.json']
```
