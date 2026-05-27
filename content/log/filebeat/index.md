+++
title = "filebeat"
date = "2026-05-28T00:01:08+08:00"
draft = false
+++

# 介绍

官方文档：https://www.elastic.co/guide/en/beats/filebeat/7.13/index.html

​	Filebeat是一种轻量型日志采集器，内置有多种模块（auditd、Apache、NGINX、System、MySQL 等等），可针对常见格式的日志大大简化收集、解析和可视化过程，只需一条命令即可。之所以能实现这一点，是因为它将自动默认路径（因操作系统而异）与 Elasticsearch 采集节点管道的定义和 Kibana 仪表板组合在一起。不仅如此，数个 Filebeat 模块还包括预配置的 Machine Learning 任务。另一点需要声明的是：根据采集的数据形式不同，形成了由多个模块组成的Beats。Beats是开源数据传输程序集，可以将其作为代理安装在服务器上，将操作数据发送给Elasticsearch，或者通过Logstash，在Kibana中可视化数据之前，在Logstash中进一步处理和增强数据。

## 特点

（1）轻量型日志采集器，占用资源更少，对机器配置要求极低。

（2）操作简便，可将采集到的日志信息直接发送到ES集群、Logstash、Kafka集群等消息队列中。

（3）异常中断重启后会继续上次停止的位置。（通过${filebeat_home}\data\registry文件来记录日志的偏移量）。

（4）使用压力敏感协议（backpressure-sensitive）来传输数据，在 logstash 忙的时候，Filebeat 会减慢读取-传输速度，一旦 logstash 恢复，则 Filebeat 恢复原来的速度。

（5）Filebeat带有内部模块（auditd，Apache，Nginx，System和MySQL），可通过一个指定命令来简化通用日志格式的收集，解析和可视化。



## Filebeat与Logstash对比

（1）Filebeat是轻量级数据托运者，您可以在服务器上将其作为代理安装，以将特定类型的操作数据发送到Elasticsearch。与Logstash相比，其占用空间小，使用的系统资源更少。

（2）Logstash具有更大的占用空间，但提供了大量的输入，过滤和输出插件，用于收集，丰富和转换来自各种来源的数据。

（3）Logstash是使用Java编写，插件是使用jruby编写，对机器的资源要求会比较高。在采集日志方面，对CPU、内存上都要比Filebeat高很多。



## 工作机制

filebeat结构：由两个组件构成，分别是inputs（输入）和harvesters（收集器），这些组件一起工作来跟踪文件并将事件数据发送到您指定的输出，harvester负责读取单个文件的内容。harvester逐行读取每个文件，并将内容发送到输出。为每个文件启动一harvester，负责打开和关闭文件，这意味着文件描述符在harvester运行时保持打开状态。如果在收集文件时删除或重命名文件，Filebeat将继续读取该文件。这样做的副作用是，磁盘上的空间一直保留到harvester关闭。默认情况下，Filebeat保持文件打开，直到达到close_inactive

关闭harvester可以会产生的结果：

- 文件处理程序关闭，如果harvester仍在读取文件时被删除，则释放底层资源。
- 只有在scan_frequency结束之后，才会再次启动文件的收集。
- 如果该文件在harvester关闭时被移动或删除，该文件的收集将不会继续

　　一个input负责管理harvesters和寻找所有来源读取。如果input类型是log，则input将查找驱动器上与定义的路径匹配的所有文件，并为每个文件启动一个harvester。每个input在它自己的Go进程中运行，Filebeat当前支持多种输入类型。每个输入类型可以定义多次。日志输入检查每个文件，以查看是否需要启动harvester、是否已经在运行harvester或是否可以忽略该文件





# 模块

## input

`Filebeat 从 7.x` 版本开始引入了新的配置方式 `filebeat.inputs`，以提供更灵活的输入配置选项，同时保留了向后兼容性。以下是 `filebeat.inputs` 和 `filebeat.prospectors` 之间的主要区别：

log（常用）：用于监视和收集文本日志文件，例如应用程序日志。

```yml
- type: log
  paths:
    - /var/log/*.log

```

stdin：用于从标准输入（stdin）收集数据。

```yml
- type: stdin
```

syslog：用于收集系统日志数据，通常是通过UDP或TCP协议从远程或本地 syslog 服务器接收。

```yml
- type: syslog
  port: 514
  protocol.udp: true
```

filestream：用于收集 Windows 上的文件日志数据。

```yml
- type: filestream
  enabled: true

```

httpjson：用于通过 HTTP 请求从 JSON API 收集数据。

```yml
- type: httpjson
  enabled: true
  urls:
    - http://example.com/api/data

```

tcp 和 udp：用于通过 TCP 或 UDP 协议收集网络数据。

```yml
- type: tcp
  enabled: true
  host: "localhost"
  port: 12345

####################
- type: udp
  enabled: true
  host: "localhost"
  port: 12345
```

总的来说，filebeat.inputs 提供了更灵活的方式来配置不同类型的输入，更容易组织和管理配置。如果您使用的是较新版本的 Filebeat，推荐使用 filebeat.inputs 配置方式。但对于向后兼容性，旧版的 filebeat.prospectors 仍然可以使用。

## processors

`processors` 是Filebeat配置中的一个部分，用于定义在事件传输到输出目标之前对事件数据进行**预处理的操作**。您可以使用 `processors` 来修改事件数据、添加字段、删除字段，以及执行其他自定义操作。以下是一些常见的 `processors` 配置示例和说明：

- **添加字段（Add Fields）**：
  可以使用 add_fields 处理器将自定义字段添加到事件中，以丰富事件的信息。例如，将应用程序名称和环境添加到事件中：

  ```yml
  processors:
    - add_fields:
    		target: "project"   # 目标字段（添加到哪个key里面），如果为空，就写到顶层
        fields:
         # key: value
          app: myapp
          env: production
          
  # 输出变成：
  {
    "project": {
      "app": "myapp",
      "env": "production"
    }
  }
  ```
  
- **删除字段（Drop Fields）**：
  使用 drop_fields 处理器可以删除事件中的指定字段。以下示例删除名为 “sensitive_data” 的字段：

  ```yml
  processors:
    - drop_fields:
        fields: ["sensitive_data"]
  ```

- **解码 JSON 字段（Decode JSON Fields）**：
  如果事件中包含JSON格式的字段，您可以使用 decode_json_fields 处理器将其解码为结构化数据。以下示例将名为 “json_data” 的字段解码为结构化数据：

  ```yml
  processors:
    - decode_json_fields:
        fields: ["message","resp"]    # 需要解码的字段
        #max_depth: 2  # 解码深度,解码所有的嵌套层级，直到没有更多的嵌套为止
        target: ""   # 为空的话，将解码后的字段合并到顶层，而不是创建一个嵌套的字段，也可以指定字段
        overwrite_keys: true    # 是否覆盖原来的key
        add_error_key: true
  
  # 对字段进行属性更改
  processors:
      - decode_json:
          fields:
            - field: "message"
              filters:
              - {target_field: "timelocal",type: "string",filter_keys: ["timestamp"]}
              - {target_field: "response_time",type: "string",filter_keys: ["response_time"]}
  ```
  
- **丢弃消息事件（drop_event）**
  
  ```yml
  # 单个条件
  processors:
  - drop_event:
      when.equals:  # 使用when条件，丢弃包含有level: "DEBUG"新的消息或者日志
        level: "DEBUG"    
  
  # 多个条件
  processors:
    - drop_event:
        when:
          or:           # 使用 or 条件来检查多个条件，丢弃包含有level: "DEBUG"或者level: "ERROR"事件消息
            - contains:
                level: "DEBUG"    
            - contains:
                level: "WARNING"
  ```
  
- #### include_fields: 指定哪些属性 (字段) 不删除，其他的都会被删除
  
  ```yml
  # 除了message属性，其他属性全部删除
  processors:
    - include_fields:
        fields: ["message"]
  ```
  
  

- **字段重命名（Rename Fields）**：
  可以使用 rename 处理器重命名事件中的字段。例如，将 “old_field” 重命名为 “new_field”：

  ```yml
  processors:
    - rename:
        fields:
          - from: old_field
            to: new_field
  
  # 也可
  # 将log下的file字段名称改为aaa
  processors:
    - rename:
        fields:
          - from: "log.file"
          - to: "log.aaa"
  ```
  
- **条件处理（Conditional Processing）**：
  使用 if 条件可以根据事件的特定字段或属性来选择是否应用某个处理器。以下示例根据事件中的 “log_level” 字段，仅在 “error” 日志级别时添加 “error” 标签：

  ```yml
  # 1、相等
  when:
    equals:
      http.response.code: 200
  	
  # 2、包含 contains condition checks if a value is part of a field.
  # fields 可以是string 或 array ; value必须是sting
  when:
    contains:
      status: "Specific error"
    
  # 3、正则匹配
  when:
    regexp:
      system.process.name: "^foo.*"
  
  # 4、范围匹配,支持：gt,gte,lt,lte，只支持integer or float 值
  when:
    range:
      http.response.code:
        gte: 400
      # 或者可以写为
      http.response.code.gte: 400
      
      # 在两个值范围内，可直接配置为：
      system.cpu.user.pct.gte: 0.5
      system.cpu.user.pct.lt: 0.8
  
  # 5. 与
  when:
    and:
      - equals:
          http.response.code: 200
      - equals:
          atus: OK
  
  # 6. 或
  when: 
    or:
      - equals:
          http.response.code: 304
      - equals:
          http.response.code: 404
  
  # 7. 非
  when:
  	not:
  		equals:
      	status: OK
  
  
  
  # 示例
  processors:
    - add_tags:
      tags: ["error"]
      when:
        equals:
            log_level: "error"
  
  ```
  
- ####  （dissect）将字段按照指定表达式来解析，将解析出来的内容添加到事件的属性中
  
  - tokenizer: 指定表达式
  
  - field: 表达式匹配哪个字段
  
  - target_prefix: 添加到哪个属性下
  
    ```yml
    # 使用空格分割message字段的内容%{xxx}代表新的key的名称,需要自定义。将key1和key2添加到事件的属性中
    # 例如日志格式为：[2023-12-15 15:10:57][WARRING][{"tid":"nacos-grpc-client-executor-nacos-cluster-svc.ywtools.svc.cluster.local-14570"}]
    processors:
    - dissect:
    	field: "message"
        tokenizer: "[%{time}][%{log_level}][%{service_name}]" 
        target_prefix: ""
    ```
  
- **时间处理（timestamp）**

  ```yml
  - timestamp:
     field: timelocal
     target_field: timestamp
     timezone: "Asia/Shanghai"
     target_layout: "UNIX_MS"
     layouts:
       - '2006-01-02 15:04:05.999'
  ```

- **多个处理器（Multiple Processors）**：
  您可以配置多个处理器，它们将按照顺序依次应用于事件数据。例如，您可以先添加字段，然后删除字段，最后重命名字段。



## output

### output.kafka

`output.kafka` 是Filebeat配置文件中的一个部分，用于配置将事件数据发送到Kafka消息队列的相关设置。以下是 `output.kafka` 部分的常见参数及其解释：

```yml
output.kafka:
  hosts: ["kafka-broker1:9092", "kafka-broker2:9092"]
  topic: "my-log-topic"
  partition.round_robin:
    reachable_only: false
  required_acks: 1
  compression: gzip
  max_message_bytes: 1000000
  
  
以下是各个参数的详细解释：

hosts（必需）：Kafka broker 的地址和端口列表。在示例中，我们指定了两个Kafka broker：kafka-broker1:9092 和 kafka-broker2:9092。Filebeat将使用这些地址来连接到Kafka集群。

topic（必需）：要发送事件到的Kafka主题（topic）的名称。在示例中，主题名称为 “my-log-topic”。Filebeat将会将事件发送到这个主题。

partition.round_robin：事件分区策略的配置。这里的配置是将事件平均分布到所有分区，不仅仅是可达的分区。reachable_only 设置为 false，表示即使分区不可达也会发送数据。如果设置为 true，则只会发送到可达的分区。

required_acks：Kafka的确认机制。指定要等待的确认数，1 表示只需要得到一个分区的确认就认为消息已经成功发送。更高的值表示更多的确认。通常，1 是常见的设置，因为它具有较低的延迟。

compression：数据的压缩方式。在示例中，数据被gzip压缩。这有助于减小传输数据的大小，降低网络带宽的使用。

max_message_bytes：Kafka消息的最大字节数。如果事件的大小超过此限制，Filebeat会将事件拆分为多个消息。

```







# 配置文件

```yml
type: log #input类型为log
enable: true #表示是该log类型配置生效
paths：     #指定要监控的日志，目前按照Go语言的glob函数处理。没有对配置目录做递归处理，比如配置的如果是：
- /var/log/* /*.log  #则只会去/var/log目录的所有子目录中寻找以".log"结尾的文件，而不会寻找/var/log目录下以".log"结尾的文件。
recursive_glob.enabled: #启用全局递归模式，例如/foo/**包括/foo, /foo/*, /foo/*/*
encoding： #指定被监控的文件的编码类型，使用plain和utf-8都是可以处理中文日志的
exclude_lines: ['^DBG'] #不包含匹配正则的行
include_lines: ['^ERR', '^WARN']  #包含匹配正则的行
harvester_buffer_size: 16384 #每个harvester在获取文件时使用的缓冲区的字节大小
max_bytes: 10485760 #单个日志消息可以拥有的最大字节数。max_bytes之后的所有字节都被丢弃而不发送。默认值为10MB (10485760)
exclude_files: ['\.gz$']  #用于匹配希望Filebeat忽略的文件的正则表达式列表
ingore_older: 0  #默认为0，表示禁用，可以配置2h，2m等，注意ignore_older必须大于close_inactive的值.表示忽略超过设置值未更新的文件或者文件从来没有被harvester收集

close_*  #close_ *配置选项用于在特定标准或时间之后关闭harvester。 关闭harvester意味着关闭文件处理程序。 如果在harvester关闭后文件被更新，则在scan_frequency过后，文件将被重新拾取。 但是，如果在harvester关闭时移动或删除文件，Filebeat将无法再次接收文件，并且harvester未读取的任何数据都将丢失。

close_inactive  #启动选项时，如果在制定时间没有被读取，将关闭文件句柄读取的最后一条日志定义为下一次读取的起始点，而不是基于文件的修改时间如果关闭的文件发生变化，一个新的harverster将在scan_frequency运行后被启动建议至少设置一个大于读取日志频率的值，配置多个prospector来实现针对不同更新速度的日志文件使用内部时间戳机制，来反映记录日志的读取，每次读取到最后一行日志时开始倒计时使用2h 5m 来表示
close_rename #当选项启动，如果文件被重命名和移动，filebeat关闭文件的处理读取
close_removed #当选项启动，文件被删除时，filebeat关闭文件的处理读取这个选项启动后，必须启动clean_removed
close_eof #适合只写一次日志的文件，然后filebeat关闭文件的处理读取
close_timeout #当选项启动时，filebeat会给每个harvester设置预定义时间，不管这个文件是否被读取，达到设定时间后，将被关闭
close_timeout #不能等于ignore_older,会导致文件更新时，不会被读取如果output一直没有输出日志事件，这个timeout是不会被启动的，至少要要有一个事件发送，然后haverter将被关闭设置0 表示不启动
clean_inactived #从注册表文件中删除先前收获的文件的状态设置必须大于ignore_older+scan_frequency，以确保在文件仍在收集时没有删除任何状态配置选项有助于减小注册表文件的大小，特别是如果每天都生成大量的新文件此配置选项也可用于防止在Linux上重用inode的Filebeat问题
clean_removed #启动选项后，如果文件在磁盘上找不到，将从注册表中清除filebeat如果关闭close removed 必须关闭clean removed
scan_frequency #prospector检查指定用于收获的路径中的新文件的频率,默认10s
tail_files： #如果设置为true，Filebeat从文件尾开始监控文件新增内容，把新增的每一行文件作为一个事件依次发送，而不是从文件开始处重新发送所有内容。
symlinks： #符号链接选项允许Filebeat除常规文件外,可以收集符号链接。收集符号链接时，即使报告了符号链接的路径，Filebeat也会打开并读取原始文件。
backoff： #backoff选项指定Filebeat如何积极地抓取新文件进行更新。默认1s，backoff选项定义Filebeat在达到EOF之后再次检查文件之间等待的时间。
max_backoff： #在达到EOF之后再次检查文件之前Filebeat等待的最长时间
backoff_factor： #指定backoff尝试等待时间几次，默认是2
harvester_limit： #harvester_limit选项限制一个prospector并行启动的harvester数量，直接影响文件打开数

tags #列表中添加标签，用过过滤，例如：tags: ["json"]
fields #可选字段，选择额外的字段进行输出可以是标量值，元组，字典等嵌套类型默认在sub-dictionary位置
filebeat.inputs:
fields:
app_id: query_engine_12
fields_under_root #如果值为ture，那么fields存储在输出文档的顶级位置

multiline.pattern #必须匹配的regexp模式
multiline.negate #定义上面的模式匹配条件的动作是 否定的，默认是false假如模式匹配条件'^b'，默认是false模式，表示讲按照模式匹配进行匹配 将不是以b开头的日志行进行合并如果是true，表示将不以b开头的日志行进行合并
multiline.match # 指定Filebeat如何将匹配行组合成事件,在之前或者之后，取决于上面所指定的negate
multiline.max_lines #可以组合成一个事件的最大行数，超过将丢弃，默认500
multiline.timeout #定义超时时间，如果开始一个新的事件在超时时间内没有发现匹配，也将发送日志，默认是5smax_procs #设置可以同时执行的最大CPU数。默认值为系统中可用的逻辑CPU的数量。name #为该filebeat指定名字，默认为主机的hostname
```



# 启动

```bash
# 进行配置文件语法检查
filebeat test config -c /path/to/your/filebeat.yml -e

# 前台启动
filebeat config -c /path/to/your/filebeat.yml
# 启动并且查看启动日志，-e 将启动信息输出到屏幕上
filebeat test config -c /path/to/your/filebeat.yml -e

# 后台启动，
# filebeat本身运行的日志默认位置${install_path}/logs/filebeat
nohup ./filebeat -e -c filebeat.yml >/dev/null 2>&1 &

```



# 实例

## 实例1：input：kafka / ouput：kafka

连接采用kerberos认证的kafka集群

```yml
filebeat.inputs:
- type: kafka
  hosts:
    - 10.14.1.162:9092
    - 10.14.3.156:9092
    - 10.14.0.151:9092
  topics: ["topic-zpa-test"]
  group_id: "filebeat"
  processors:
  # 将message中的json数据直接打散
  - decode_json_fields: 
      fields: ["message"]
      target: ""
      overwrite_keys: true
      add_error_key: true

output.kafka:
  version: "1.1.0"
  enabled: true
  hosts: ["10.200.37.182:21007","10.200.37.69:21007","10.200.37.72:21007"]
  topic: 'webservice'
  # fail_to_topic: '%{[@logErrTopic]}'
  worker: 1
  bulk_max_size: 16000
  compression: none
  # 采用kerberos认证
  kerberos:
    enabled: true
    auth_type: keytab
    service_name: kafka
    username: kafka_poisson
    keytab: /opt/kafka_2.12-3.3.1/cert/user.keytab
    config_path: /opt/kafka_2.12-3.3.1/cert/krb5.conf
    realm: AIOPS_APAAS_MRS_STREAM_302.COM
    domain: hadoop.aiops_apaas_mrs_stream_302.com

logging.to_files: true
logging.to_syslog: false
logging.level: info
logging.files:
  path: /opt/filebeat-huawei/log/
  name: filebeat.log

max_procs: 1
path.data: /opt/filebeat-huawei/data/
queue.mem.events: 64000
queue.mem.flush.min_events: 32000
```























