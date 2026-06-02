+++
title = "graylog"
date = "2026-05-28T00:01:08+08:00"
draft = false
weight = 3
+++

# 介绍

Graylog - 日志监控系统，官文：https://docs.graylog.org/docs

Graylog 是一个开源的日志聚合、分析、审计、展现和预警工具。在功能上来说，和 ELK 类似，但又比 ELK 要简单很多。依靠着更加简洁，高效，部署使用简单的优势很快受到许多人的青睐。当然，在扩展性上面确实没有比 ELK 好，但是其有商业版本可以选择。

部署 graylog 最简单的架构就是单机部署，复杂的也是部署集群模式，架构图示如下所示。可以看到其中包含了三个组件，分别是 Elasticsearch、MongoDb 和 Graylog。

- Elasticsearch 用来持久化存储和检索日志文件数据(IO 密集)
- MongoDb 用来存储关于 Graylog 的相关配置
- Graylog 来提供 Web 界面和对外接口的(CPU 密集)。

<img src="./images/image-20240521205326781.png" alt="image-20240521205326781" style="zoom:50%;" />

![image-20240522160447783](./images/image-20240522160447783.png)

生产环境架构：

![image-20240521210940478](./images/image-20240521210940478.png)

Graylog 组件功能

| 编号 | 组件名称      | 功能介绍           | 主要特点                                                     |
| ---- | ------------- | ------------------ | ------------------------------------------------------------ |
| 1    | Dashboards    | 数据面板固定展示   | 主要是用来保存特定搜索条件的数据面板                         |
| 2    | Searching     | 日志信息条件搜索   | 关键字搜索、时间搜索、搜索保存、创建面板、分组查询、结果导出、查询高亮显示、自定义时间 |
| 3    | Alert         | 设置告警提示方式   | 支持邮件告警、HTTP 回调和自定义脚本触发                      |
| 4    | Inputs        | 日志数据抓取接收   | 部署 Sidercar 主动抓取或使用其他服务被动上报                 |
| 5    | Extractors    | 日志数据格式转换   | json 解析、kv 解析、时间戳解析、正则解析                     |
| 6    | Streams       | 日志信息分类分组   | 设置日志分类条件并发送到不同的索引文件中去                   |
| 7    | Indices       | 持久化数据存储     | 设置数据存储性能                                             |
| 8    | Outputs       | 日志数据的转发     | 解析的 Stream 发送到其他 Graylog 集群或服务                  |
| 9    | Pipelines     | 日志数据的过滤     | 建立数据清洗的过滤规则、字段添加删除、条件过滤、自定义函数等 |
| 10   | Sidecar       | 轻量级的日志采集器 | 相当于 C/S 模式；大规模时使用                                |
| 11   | Lookup Tables | 服务解析           | 基于 IP 的 Whois 查询和基于来源 IP 的情报监控                |
| 12   | Geolocation   | 可视化地理位置     | 实现基于来源 IP 的情报监控                                   |

<img src="./images/image-20240522134617201.png" alt="image-20240522134617201" style="zoom: 50%;" />

Graylog 通过 Input 搜集日志，每个 Input 单独配置 Extractors 用来做字段转换。Graylog 中日志搜索的基本单位是 Stream，每个 Stream 可以有自己单独的 Elastic Index Set，也可以共享一个 Index Set。

Extractor 在 System/Input 中配置。Graylog 中很方便的一点就是可以加载一条日志，然后基于这个实际的例子进行配置并能直接看到结果。内置的 Extractor 基本可以完成各种字段提取和转换的任务，但是也有些限制，在应用里写日志的时候就需要考虑到这些限制。Input 可以配置多个 Extractors，按照顺序依次执行。

系统会有一个默认的 Stream，所有日志默认都会保存到这个 Stream 中，除非匹配了某个 Stream，并且这个 Stream 里配置了不保存日志到默认 Stream。可以通过菜单 Streams 创建更多的 Stream，新创建的 Stream 是暂停状态，需要在配置完成后手动启动。Stream 通过配置条件匹配日志，满足条件的日志添加 stream ID 标识字段并保存到对应的 Elastic Index Set 中。

Index Set 通过菜单 System/Indices 创建。日志存储的性能，可靠性和过期策略都通过 Index Set 来配置。性能和可靠性就是配置 Elastic Index 的一些参数，主要参数包括，Shards 和 Replicas。

除了上面提到的日志处理流程，Graylog 还提供了 Pipeline 脚本实现更灵活的日志处理方案。这里不详细阐述，只介绍如果使用 Pipelines 来过滤不需要的日志。下面是丢弃 level > 6 的所有日志的 Pipeline Rule 的例子。从数据采集(input)，字段解析(extractor)，分流到 stream，再到 pipeline 的清洗，一气呵成，无需在通过其他方式进行二次加工。



Sidecar 是一个轻量级的日志采集器，通过访问 graylog 进行集中式管理，支持 linux 和 windows 系统。Sidecar 守护进程会定期访问 graylog 的 REST API 接口获取 Sidecar 配置文件中定义的标签(tag) ，Sidecar 在首次运行时会从 graylog 服务器拉取配置文件中指定标签(tag) 的配置信息同步到本地。目前 Sidecar 支持 NXLog，Filebeat 和 Winlogbeat。他们都通过 graylog 中的 web 界面进行统一配置，支持 Beats、CEF、Gelf、Json API、NetFlow 等输出类型。Graylog 最厉害的在于可以在配置文件中指定 Sidecar 把日志发送到哪个 graylog 群集，并对 graylog 群集中的多个 input 进行负载均衡，这样在遇到日志量非常庞大的时候，graylog 也能应付自如。



优缺点

**Graylog优点**

1. 一体化方案，安装方便，不像ELK有3个独立系统间的集成问题。
2. 采集原始日志，并可以事后再添加字段，比如http_status_code，response_time等等。
3. 自己开发采集日志的脚本，并用curl/nc发送到Graylog Server，发送格式是自定义的GELF，Flunted和Logstash都有相应的输出GELF消息的插件。自己开发带来很大的自由度。实际上只需要用inotify_wait监控日志的MODIFY事件，并把日志的新增行用curl/nc发送到Graylog Server就可。
4. 搜索结果高亮显示，就像google一样。
5. 搜索语法简单，比如： source:mongo AND reponse_time_ms:>5000 ，避免直接输入elasticsearch搜索json语法搜索条件可以导出为elasticsearch的搜索json文本，方便直接开发调用elasticsearch rest api的搜索脚本。

**Graylog缺点**

1. 控制台操作页面是英文的，针对国内开发使用者使用起来不方便，还得额外汉化，汉化可能失败
2. 使用网络传输，可能会占用项目网络



# 安装



# 使用

服务正常启动后就可以通过[http://ip:9000](http://ip:9000/) 进行访问graylog的Web界面，默认用户admin/admin。

![image-20240522144559187](./images/image-20240522144559187.png)

Graylog的日志收集通过定义input来完成，在Graylog的Web管理页面的System tab下可以选择定义input来对日志进行收集

![image-20240522144628645](./images/image-20240522144628645.png)

进入input页面后选择input的类型，比如定义GELF UDP的input:

![image-20240522144633633](./images/image-20240522144633633.png)

选择完成后点击 “Lanch new input”，就会进入详细的input配置，配置完成后保存就可以了

![image-20240522144745309](./images/image-20240522144745309.png)

保存后一切正常的话，input就会进入RUNNING状态，这时就可以往这个input里面发送数据了，点击“Stop input”，input就会停止，数据的接收也会停止，“Stop input”会变成“Start input”，需要接受数据的时点击启动就可以了。

![image-20240522144802308](./images/image-20240522144802308.png)