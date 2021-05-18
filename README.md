# Elasticsearch

## 1. CentOS-Elasticsearch集群部署

### 1.1 前期工作：

- Elasticsearch是基于Java环境开发的一款文档存储的快速搜索工具，因此我们在部署集群的时候，必须得确保我们的系统具备Java环境。

```bash
[root@ecs-564d ~]# java -version
openjdk version "1.8.0_292"
OpenJDK Runtime Environment (build 1.8.0_292-b10)
OpenJDK 64-Bit Server VM (build 25.292-b10, mixed mode)
```

- 下载安装Elasticsearch，

  在CentOS下可通过wget命令直接通过网络请求下载源文件并解压即可

  ```bash
  wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.12.1.tar.gz
  
  tar -zxvf elasticsearch-7.12.1.tar.gz /filepath #指定你想存放的位置
  ```

### 1.2 配置Elasticsearch集群：

1. #### 创建节点文件夹

   因为我们是在单机上部署的伪集群，当然有条件的可以利用多个服务器进去部署，相关的配置方法是类似的。单机环境下我们将通过端口号来区分我们的节点：`9201`、`9202`、`9203`创建如下三个个文件夹：（WinSCP）

   ![image-20210511102658433](\ES-ljm\image-20210511102658433.png)

   如图名称可自取。在每一个文件夹下面我们将下载的elasticsearch的文件全数拷贝至这两文件夹

   ```bash
   cp elasticsearch-7.12.1 elasticsearch_node1
   cp elasticsearch-7.12.1 elasticsearch_node2
   cp elasticsearch-7.12.1 elasticsearch_node3
   ```

2. #### 修改配置文件（重要）

   切换目录到对应节点的config目录

   ```bash
   cd /config
   vim elasticsearch.yml
   ```

   在编辑的文件中添加如下的配置：

   ```yaml
   cluster.name: myesc #集群的名称
   node.name: node1 #节点名称
   node.master: true #是否为主节点
   node.data: true #是否负责数据查询导入等
   path.data: /usr/local/elasticsearch-cluster/elasticsearch-node1/elasticsearch-7.12.1/data #数据存放目录
   path.logs: /usr/local/elasticsearch-cluster/elasticsearch-node1/elasticsearch-7.12.1/logs #日志文件输出目录
   bootstrap.memory_lock: false #是否锁住内存，避免交换(swapped)带来的性能损失,默认值是: false
   bootstrap.system_call_filter: false
   network.host: 0.0.0.0 #允许访问的IP地址
   http.port: 9201 #端口
   transport.tcp.port: 9301 #转换端口
   discovery.seed_hosts: ["127.0.0.1:9301","127.0.0.1:9302","127.0.0.1:9303"] #这是7.x之后的discover.zen.ping.unicast.hosts--写入候选的节点地址
   #集群初始化主节点，初次启动主节点的时候必须配置在你想要的主节点下面
   cluster.initial_master_nodes: ["node1"]
   http.cors.enabled: true #允许跨域（在使用插件监控elasticsearch集群的时候需要配置的）
   http.cors.allow-origin: "*" #"*"允许所有的域名
   ```

3. #### 修改系统配置

   - 因为Java是基于jvm虚拟机运行的，而且elasticsearch需要消耗内存，当我们在单机下配置的时候需要注意的是。为每一个节点分配的xms内存应该在一般的基础上还得考虑节点的个数均匀分配，以免后续不必要的错误，可能不能启动其他的节点。

   ```bash
   vim config/jvm.options
   # 按需修改如下内存大小即可,因为我的服务器内存较小所以分配的较少，可根据自己的服务器来设置，一般为一半
   -Xms256m
   -Xmx256m
   ```

   -  修改用户基础允许的文件描述符大小（每个进程最大同时打开文件数大小）

   Lucene 使用了 *大量的* 文件。 同时，Elasticsearch 在节点和 HTTP 客户端之间进行通信也使用了大量的套接字（注：sockets）。 所有这一切都需要足够的文件描述符。特别是节点将文档进行大量的索引的时候（我的理解是，索引的时候需要明确知道文档的位置，Linux操作系统下面通过文件描述符来进行数据的交换等操作）

   ```bash
   vim /etc/security/limits.conf
   ```

   添加如下

   ```bash
   # * 表示所有的用户
   * soft nofile 65535
   * hard nofile 65536
   * soft nproc 102400
   * soft memlock unlimited #内存锁定不限制
   * hard memlock unlimited
   ```

   - 最大虚拟内存

   ```bash
   vim /etc/sysctl.conf
   ```

   添加

   ```
   vm.max_map_count=655360
   ```

   最后使系统配置立即生效

   ```bash
   sysctl -p
   #如果配置没有生效，建议将服务器重新启动即可。
   ```

4. #### 启动elasticsearch

   由于elasticsearch默认不能在root用户下启动，我们需要创建一个普通用户去启动

   ```bash
   su elastic #自己随意创建一个用户
   ```

### 1.3 插件连接elasticsearch

- elasticsearch-head

  该插件对于查看集群的节点以及节点索引信息，个人感觉很好，页面简洁易懂上手很快。特别是可以在chrome浏览器可直接安装扩展程序：https://chrome.google.com/webstore/detail/elasticsearch-head/ffmkiejjmecolpfloofpjologoblkegm

  直接通过ip以及port连接即可，如图：

  ![image-20210511111603487](\ES-ljm\image-20210511111603487.png)

- Kibana

  这是elastic官方的插件，需要在官网去下载源程序：https://www.elastic.co/cn/downloads/kibana，下载并解压，进入解压完的配置文件夹config，并配置kibana.yml

  ```bash
  elasticsearch.hosts: ["http://ESIP:9201, http://ESIP:9202"]
  #配置远程的ES集群
  ```

  ```bash
  #运行bin文件夹下面的kibana脚本启动文件即可
  ```

  打开浏览器输入：http://localhost:5601/ 进入kibana界面

  ![image-20210511112437256](\ES-ljm\image-20210511112437256.png)

上图标注的地方是我们比较常用的部分Dev Tools通过api形式可直接操作我们的ES数据库，其中还提供代码补全的功能。最好就是kibana操作ES，再用elasticsearch-head验证结果，head插件还可以每个文档的详细数据

## 2. Elasticsearch集群原理

### 2.1 分布式特性

- 分配文档到不同的容器 或 *分片*（主分片） 中，文档可以储存在一个或多个节点中
- 按集群节点来均衡分配这些分片，从而对索引和搜索过程进行负载均衡
- 复制每个分片以支持数据冗余（分片副本），从而防止硬件故障导致的数据丢失
- 将集群中任一节点的请求路由到存有相关数据的节点（路由配置）
- 集群扩容时无缝整合新节点，重新分配分片以便从离群节点恢复（横向扩容）

![](\ES-ljm\Snipaste_2021-04-30_11-08-18.png)

一个elasticsearch集群里面包含了多个节点， 一个节点中有多个索引--索引指向了分片，而分片是一个工作单位，一个分片存储的是全部数据的一部分，所有分片加一起的数据才是完整的全部数据。每一个主分片中都可以指定一定数量的副本。（主分片不一定全部都在一个节点里面） 索引内任意一个文档都归属于一个主分片，所以主分片的数目决定着索引能够保存的最大数据量。

### 2.2 分布式文档存储

 路由一个文档到一个分片：

协调节点：将请求全部发送到某个的节点

```mathematica
//路由的分片位置（0-----number_of_primary_shards-1）
shard = hash(routing) % number_of_primary_shards
//routing: 默认为_id
//number_of_primary_shards: 主分片数量
```

## 3. Elasticsearch -常用api操作

- 索引库（indices)	indices是index的复数，代表许多的索引，
- 类型（type）	类型是模拟mysql中的table概念，一个索引库下可以有不同类型的索引，比如商品索引，订单索引，其数据格式不同。不过这会导致索引库混乱，因此7.x版本中会移除这个概念
- 文档（document）	存入索引库原始的数据。比如每一条商品信息，就是一个文档
- 字段（field）	文档中的属性

### 3.1 创建索引

```bash
PUT http://ESIP:port/indexName
```

请求参数：

```json
{
    //指定在该集群下的分片情况，以及副本情况
    "settings": {
        "number_of_shards": 3, //主分片数量，默认为5，
        "number_of_replicas": 1 // 副本数量，默认为1.（每一个主分片的副本数量）
    },
    //映射配置： 字段的数据类型、属性、是否索引、是否存储
    "mapping": {
        "properties": {
            ***
        }
    }
}
```

### 3.2 添加/修改文档（数据）index

indexName + type + id唯一确定一个文档

```bash
PUT http://ESIP:port/indexName/id
```

请求参数示例：

```json
{
    "name": "快乐星球"
    *** //你要添加/修改的其他属性 ---对应Java实体类的属性
}
```

如果在该索引，类型和id对应的文档存在，则更新， 不存在则添加到该索引下。

### 3.3 GET/DELETE文档

```bash
#获取文档
GET http://ESIP:port/indexName/id
#删除文档
DELETE http://ESIP:port/indexName/id
```

### 3.4 Update

```bash
POST http://ESIP:port/indexName/id/_update
```

```json
//修改的字段参数
{
    "name": "快乐老家"
}
```

### 高级

### 3.5 并发控制（version）

```bash
POST http://ESIP:port/indexName/type/id?version=2
```

```json
{
	"name": "大哥"
}
//该api可以通过控制文档的版本号来更新（避免冲突）
```

当文档被修改时，版本号递增（乐观锁机制），上面的例子说明：指定当我们数据版本号是2时才更新成功，否则拒绝

### 3.6 批量操作（Bulk）

运行在单个操作中进行多次的以上介绍的简单操作：create、index、update、 delete

```json
{ action: { metadata }}\n
{ request body        }\n
{ action: { metadata }}\n
{ request body        }\n
```

- 必须要在每个操作以及请求体后面加上换行符合"\n"
- delete操作没有请求体，后面紧接着另外的操作

```json
POST /_bulk
{ "delete": { "_index": "website", "_type": "blog", "_id": "123" }} 
{ "create": { "_index": "website", "_type": "blog", "_id": "123" }}
{ "title":    "My first blog post" }
{ "index":  { "_index": "website", "_type": "blog" }}
{ "title":    "My second blog post" }
{ "update": { "_index": "website", "_type": "blog", "_id": "123", "_retry_on_conflict" : 3} }
{ "doc" : {"title" : "My updated blog post"} } 
```

#### 6.1 bulk请求格式说明：（最小代价）

如果不用换行符来区分每一个操作请求，即所有的请求被放在一个json数组里面，然后解析json为数组，确定请求属于哪个分片，为每个分片创建一个请求数组， 再将数组序列化发送到每个副本分片-----**需要大量的 RAM 来存储原本相同的数据的副本**

elasticsearch利用这种格式，将原始请求直接正确转发到正确的分片，避免复制原始数据转发到每一个分片（从网络缓冲区取）

### 3.7 搜索

#### 3.7.1  复杂条件查询：

```txt
GET /IndexName/_search
```

```json
{
    "query": {
        "bool": {
            #1.must
            "must": [{
                "match": {}
            }],
            "filter": [{
                "range": {}
            }]
        }
    }
}
```

- *过滤器* range用于执行范围查询， must用于filed==value或者like %*%操作

#### 3.7.2  短语搜索

精确匹配一系列单词或者短语。

```json
GET /IndexName/_search
{
    "query": {
        "match_phrase": {
            //字段
            "Field": "短语"
        }
    },
    //高亮短语片段
    "highlight": {
        "fileds": {
            //字段
            "FIELD"
        }
    }
}
```

#### 3.7.3  聚合分析

aggregation(聚合)类似sql中的GROUP BY，允许基于数据生成一些精细的分析结果。

```json
GET /IndexName/_search
{
  "query": {
    //条件
    "match": {
      *****
    }
  },
  "aggs": {
    "聚合名称": {
      //terms聚合方法（出现次数最多）
      "terms": {
        "field": "yourfield"
      },
      //分级聚合
      "aggs": {
        "聚合名称": {
          //平均值聚合 avg
          "avg": {
            "field": "yourfield"
          }
        }
      }
    }
  }
}
```

#### 3.7.4  分页

```json
GET /IndexName/_search?size=5&from10
```

**`size`**

显示应该返回的结果数量，默认是 `10`

**`from`**

显示应该跳过的初始结果数量，默认是 `0`

#### 3.7.5 结构化搜索（filter）

首先考虑**过滤器**-----理论上（非评分计算相关）

核心实际是采用一个 bitset 记录与过滤器匹配的文档。Elasticsearch 积极地把这些 bitset 缓存起来以备随后使用。一旦缓存成功，bitset 可以复用 *任何* 已使用过的相同过滤器，而无需再次计算整个过滤器。

- *结构化搜索（Structured search）* 是指有关探询那些具有内在结构数据的过程。比如日期、时间和数字都是结构化的：它们有***精确的格式***，我们可以对这些格式进行逻辑操作。比较常见的操作包括比较数字或时间的范围，或判定两个值的大小。

- 结构化查询中，我们得到的结果 *总是* 非是即否，要么存于集合之中，要么存在集合之外

##### 3.7.5.1 精确值查询：

`term` 和 `terms` 是 *必须包含（must contain）* 操作，而不是 *必须精确相等（must equal exactly）* 。对单个词项进行操作，且在倒排索引中查找`准确词项`

（math等高层查询的全文查询，（analyzed），将字符串传入到分析器生成词项列表，再对每个词项进行底层查询，将合并后的结果传回（带相关度评分））

`bool`匹配器，bool体里面还能包含其他的bool匹配器等,这其中还能组合出更多的组合可能来适用负责的条件查询。

即整个字段完全相等--------增加并索引另一个字段， 这个字段用以存储该字段包含词项的数量，然后构造constant_score查询

##### 3.7.5.2 范围查询

range过滤器：数字以及时间格式的都可用range进行范围查询。同样elasticsearch支持字符串的范围查询--字典顺序。但是字符串的范围查询不推荐对唯一字符过多的内容进行比较查询。

#### 3.7.6 全文搜索（查询、搜索）

##### 3.7.6.1 倒排索引

Elasticsearch 使用一种称为 *倒排索引* 的结构，它适用于快速的全文搜索。一个倒排索引由文档中所有不重复词的列表构成，对于其中每个词，有一个包含它的文档列表。为了创建倒排索引，我们首先将每个文档的 filed的content域拆分成单独的 词（我们称它为 `词条` 或 `tokens` ），创建一个包含所有不重复词条的排序列表，然后列出每个词条出现在哪个文档。

Elasticsearch 基于 Lucene, 引入了 *按段搜索* 的概念。 每一 *段* 本身都是一个倒排索引， 但 *索引* 在 Lucene 中除表示所有 *段* 的集合外， 还增加了 *提交点* 的概念 — 一个列出了所有已知段的文件，新的文档首先被添加到内存索引缓存中，然后写入到一个基于磁盘的段。

![img](https://upload-images.jianshu.io/upload_images/6807865-bd90dd902aecb414.png?imageMogr2/auto-orient/strip|imageView2/2/format/webp)

1）数据先写入内存buffer，在写入buffer的同时将数据写入translog日志文件，注意：此时数据还没有被成功es索引记录，因此无法搜索到对应数据；

2）如果buffer快满了或者到一定时间，es就会将buffer数据refresh到一个新的segment file中，但是此时数据不是直接进入segment file的磁盘文件，而是先进入os cache的。这个过程就是refresh。
每隔1秒钟，es将buffer中的数据写入一个新的segment file，因此每秒钟会产生一个新的磁盘文件segment file，这个segment file中就存储最近1秒内buffer中写入的数据。
操作系统中，磁盘文件其实都有一个操作系统缓存os cache，因此数据写入磁盘文件之前，会先进入操作系统级别的内存缓存os cache中。

一旦buffer中的数据被refresh操作，刷入os cache中，就代表这个数据就可以被搜索到了。

这就是为什么es被称为准实时：因为写入的数据默认每隔1秒refresh一次，也就是数据每隔一秒才能被 es 搜索到，之后才能被看到，所以称为准实时。

只要数据被输入os cache中，buffer就会被清空，并且数据在translog日志文件里面持久化到磁盘了一份，此时就可以让这个segment file的数据对外提供搜索了。

3）重复1~2步骤，新的数据不断进入buffer和translog，不断将buffer数据写入一个又一个新的segment file中去，每次refresh完，buffer就会被清空，同时translog保留一份日志数据。随着这个过程推进，translog文件会不断变大。当translog文件达到一定程度时，就会执行commit操作。

4）commit操作发生第一步，就是将buffer中现有数据refresh到os cache中去，清空buffer。

5）将一个 commit point 写入磁盘文件，里面标识着这个 commit point 对应的所有 segment file，同时强行将 os cache 中目前所有的数据都 fsync 到磁盘文件中去。

6）将现有的translog清空，然后再次重启启用一个translog，此时commit操作完成。

![](\ES-ljm\绘图7.jpg)

##### 3.7.6.2 映射（Mapping）

描述数据在每个字段内如何存储

###### 精确值、全文

`string` 域映射的两个最重要属性是 `index` 和 `analyzer` 。

`index` 属性控制怎样索引字符串。它可以是下面三个值：(7.x版本index的值改为true/false，表示是否索引，能不能被搜索到。 而其是否分析字符串则通过mapping的type来决定“keyword"对应not_analyzed, "text"对应analyzed)

- **`analyzed`**

  首先分析字符串，然后索引它。换句话说，以全文索引这个域。

- **`not_analyzed`**

  索引这个域，所以它能够被搜索，但索引的是精确值。不会对它进行分析。

- **`no`**

  不索引这个域。这个域不会被搜索到。

##### 3.7.6.3 分析（Analysis）

全文是如何处理使得可搜索

*分析* 包含下面的过程：

- 首先，将一块文本分成适合于倒排索引的独立的 *词条* ，
- 之后，将这些词条统一化为标准格式以提高它们的“可搜索性”.

###### **字符过滤器**

首先，字符串按顺序通过每个 *字符过滤器* 。他们的任务是在分词前整理字符串。一个字符过滤器可以用来去掉HTML，或者将 `&` 转化成 `and`。

###### **分词器**

其次，字符串被 *分词器* 分为单个的词条。一个简单的分词器遇到空格和标点的时候，可能会将文本拆分成词条。

###### **Token 过滤器**（词条过滤器）

最后，词条按顺序通过每个 *token 过滤器* 。这个过程可能会改变词条（例如，小写化 `Quick` ），删除词条（例如， 像 `a`， `and`， `the` 等无用词），或者增加词条（例如，像 `jump` 和 `leap` 这种同义词）。

###### 创建自定义的分析器

```json
PUT /my_index
{
    "settings": {
        "analysis": {
            "char_filter": { ... custom character filters ... }, //字符过滤器
            "tokenizer":   { ...    custom tokenizers     ... },//分词器
            "filter":      { ...   custom token filters   ... },//词单元过滤器
            "analyzer":    { ...    custom analyzers      ... } //分析器整合
        }
    }
}
```

```json
PUT /my_index
{
    "settings": {
        "analysis": {
            "char_filter": {
                //过滤html字符&为and
                "&_to_and": {
                    "type":       "mapping",
                    "mappings": [ "&=> and "]
            }},
            "filter": {
                //tokens词条过滤停用词
                "my_stopwords": {
                    "type":       "stop",
                    "stopwords": [ "the", "a" ]
            }},
            "analyzer": {
                "my_analyzer": {
                    "type":         "custom",
                    "char_filter":  [ "html_strip", "&_to_and" ],
                    "tokenizer":    "standard",
                    "filter":       [ "lowercase", "my_stopwords" ]
            }}
}}}
```

## 4. logback+ELK（日志记录存储）

springboot框架本身就是默认的日志打印为logbach,所以该框架是可以适应于普遍的springboot项目。基本框架类似于下图：

![](\ES-ljm\绘图2.png)

Logstash能够动态地采集、转换和传输数据，不受格式或复杂度的影响。在该框架下Logstash会捕获项目运行时logback产生的日志记录，获取日志记录数据，并利用其过滤器将日志数据进行实时解析和转换数据，将其转换成目的存储库的通用数据格式，最后输出到目标存储数据库ES。Logstash的具体结构则为`input`->`filter` ->`output`

### 4.1 下载安装logstash

以下为了方便我在window系统下载安装的logstash和kibana。如在Linux系统下其配置是类似的，只是执行的命令不一样。从官网下载logstash:https://www.elastic.co/cn/downloads/logstash, 下载完成后解压到目的目录。

### 4.2 配置logstash

1. 配置logstash就是配置其输入、过滤器规则、以及输出目标

2. 切换至`/config`文件夹

3. 创建logstash.conf文件，并写入：

   ```ruby
   #输入
   input {
     tcp {
         #定义链接logstash的地址及端口
       host => "127.0.0.1"
       port => 9250
       mode => "server"
       tags => ["tags"]
       codec => json_lines
       }
   }
   #过滤
   filter{
       #ruby编码，由于logstash默认时区为UTC，于我们所在地区相差八小时
       #编码则通过增加一个字段对自动生成的时间戳进行运算为正确的时间
       #避免输出按天滚动的内容不符
     date {
       match => ["message","UNIX_MS"]
       target => "@timestamp"   
     }
    ruby { 
      code => "event.set('timestamp', event.get('@timestamp').time.localtime + 8*60*60)" 
    }
    ruby {
      code => "event.set('@timestamp',event.get('timestamp'))"
    }
    mutate {
      remove_field => ["timestamp"]
      remove_field => ["tags"]
      remove_field => ["@version"]
      remove_field => ["port"]
    }
   }
   #输出
   output {
     #控制台直接输出
     stdout{
       codec => rubydebug
     }
     #输出的存储数据库地址
     elasticsearch{
   	hosts => ["ESIP:9201", "ESIP:9202"]
   	action => "index"
         #按天滚动创建索引
   	index => "yt-logback-%{+YYYY.MM.dd}"
     }
   }
   ```

4. 修改pipelines.yml: logstash是利用管道方式进行通信的，一个logstash可以连接多个管道，每个管道可配置隔离，虽然logstash本身没有集群但是其同样可以利用多管道的功能实现分布式的日志管理。

   ![](C:\Users\l50018000\Desktop\ES图\2018062921260744.png)

   编辑写入：

   ```yaml
    - pipeline.id: test1
      pipeline.workers: 1 #上图pipeline线程数
      pipeline.batch.size: 1 #上图中batcher一次批量获取队列里面的文档数
      path.config: "/edgeDownload/logstash-7.12.1-windows-x86_64/logstash-7.12.1/config/logstash.conf" #设置配置文件的目录，目的为获取input,filte,output的配置
   ```

5. 运行启动`logstash.bat`

   ![image-20210511153735076](\ES-ljm\image-20210511153735076.png)

   如图出现Pipeline started则说明启动成功

![image-20210511153754857](\ES-ljm\image-20210511153754857.png)

### 4.3 配置springboot项目

1. logback需要连接logstash需要导入相关的maven来引入logstash的相关组件，即需要将logback的日志输出到logstash的必要组件

   ```xml
   <dependency>
   	<groupId>net.logstash.logback</groupId>
       <artifactId>logstash-logback-encoder</artifactId>
       <version>6.6</version>
   </dependency>
   ```

2. 新建logback-spring.xml，如果项目已经将logback作为日志输出的组件，即可直接添加如下配置

   ```xml
    <?xml version="1.0" encoding="UTF-8"?>
   <configuration scan="true" scanPeriod="60 seconds" debug="true">
       <!-- 获取ip地址 --> 
   <conversionRule conversionWord="ip" converterClass="com.robin.elkdemo.config.MyIpConfig"></conversionRule>
   <appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
         <!-- 远程logstash地址（即上面logstash.conf配置的输入tcp配置） -->   
        <destination>127.0.0.1:9250</destination>
           <!-- 自定义输出到logstash的json模板格式-->
           <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder" charset="UTF-8">
               <providers>
                   <pattern>
                       <pattern>
                           {
                           <!-- 该ip的获取需自定义实现获取-->
                           "host": "%ip",
                           "totalMes": "%d{yyyy-MM-dd HH:mm:ss} %-5level %logger{50} - %msg",
                           "level": "%level",
                           "pid": "${PID:-}",
                           "class": "%logger",
                           "message": "%message",
                           "stack_trace": "%exception{40}"
                           }
                       </pattern>
                   </pattern>
               </providers>
           </encoder>
       </appender>
   
   <root level="INFO">
   	<appender-ref ref="LOGSTASH" />
   </root>
   </configuration>
   ```

3. MyIpConfig

   ```java
   public class MyIpConfig extends ClassicConverter {
       @Override
       public String convert(ILoggingEvent iLoggingEvent) {
           try{
               return InetAddress.getLocalHost().getHostAddress();
           }catch (UnknownHostException e){
               e.printStackTrace();
           }
           return null;
       }
   }
   ```

### 4.4 运行效果

![image-20210511155834700](\ES-ljm\image-20210511155834700.png)

如图输入的日志记录进入logstash输出在命令行的已经是过滤之后的。因此在实际应用中我们可以利用过滤器对数据进行定制化输出。

## 5. Mysql+ELK（准实时同步数据库）

### 5.1 准备工作

logstash基于Java代码编写的，其同步数据库的原理是在数据库连接池中通过jdbc连接数据库，并执行相关select语句获取符合条件的表格数据，再输出到ES数据库当中，（一表对应一个索引），多表同步则配置多个jdbc输入，如图：

![image-20210511163514242](\ES-ljm\image-20210511163514242.png)

### 5.2 配置同步数据库（MySQL）

1. 在config同级目录下新建文件夹`mysql`

2. 下载对应mysql版本的mysql-connector-java-8.0.23.jar放在该目录下面

3. 新建需要的test.sql文件以及存放上一次最后运行的文件last_run.txt

4. 编辑test.sql文件(根据自己的表来写), :sql_last_value就是last_run.txt存的数值。（内置函数）

   ```sql
   SELECT * FROM env
   WHERE last_update_time > :sql_last_value 
   ORDER BY last_update_time ASC
   ```

5. 新建配置文件`jdbc.conf`,并写入：

   ```ruby
   input {
   	stdin {}
   	jdbc {
           # 设置该jdbc连接的名称，在配置多表同步的时候需要唯一的名称
   		type => "jdbc"
   		 # 数据库连接地址
   		jdbc_connection_string => "jdbc:mysql://MYSQLIP:3306/yuntan?characterEncoding=UTF-8&autoReconnect=true"
   		 # 数据库连接账号密码；
   		jdbc_user => "****"
   		jdbc_password => "******"
   		 # MySQL依赖包路径；
   		jdbc_driver_library => "D:/edgeDownload/logstash-7.12.1-windows-x86_64/logstash-7.12.1/mysql/mysql-connector-java-8.0.23.jar"
   		 # MySQL在8.0的需要以下的驱动器
   		jdbc_driver_class => "com.mysql.cj.jdbc.Driver"
   		 # 数据库重连尝试次数
   		connection_retry_attempts => "3"
   		# 数据库时区设置 默认为UTC
   		jdbc_default_timezone => "Asia/Shanghai"
   		#使用本地时区为local，否则sql_last_value如果是timestamp，时间会提前8小时
           #值可以是：utc，local，默认值为 "utc"
   		plugin_timezone => "local"
   		 # 判断数据库连接是否可用，默认false不开启
   		jdbc_validate_connection => "true"
   		 # 数据库连接可用校验超时时间，默认3600S
   		jdbc_validation_timeout => "3600"
   		 # 开启分页查询（默认false不开启）；
   		jdbc_paging_enabled => "true"
   		 # 单次分页查询条数（默认100000,若字段较多且更新频率较高，建议调低此值）；
   		jdbc_page_size => "500"
   		 # statement为查询数据sql，如果sql较复杂，建议配通过statement_filepath配置sql文件的存放路径；
   		 # sql_last_value为内置的变量，存放上次查询结果中最后一条数据tracking_column的值，此处即为ModifyTime；
   		statement_filepath => "D:/edgeDownload/logstash-7.12.1-windows-x86_64/logstash-7.12.1/mysql/test.sql"
   		 # 是否将字段名转换为小写，默认true（如果有数据序列化、反序列化需求，建议改为false）；
   		lowercase_column_names => false
   		 # Value can be any of: fatal,error,warn,info,debug，默认info；
   		sql_log_level => warn
   		 # 是否记录上次执行结果，true表示会将上次执行结果的tracking_column字段的值保存到last_run_metadata_path指定的文件中；
   		record_last_run => true
   		 # 需要记录查询结果某字段的值时，此字段为true，否则默认tracking_column为timestamp的值；
   		use_column_value => true
   		 # 需要记录的字段，用于增量同步，需是数据库字段
   		tracking_column => "last_update_time"
   		 # Value can be any of: numeric,timestamp，Default value is "numeric"
   		tracking_column_type => timestamp
   		 # record_last_run上次数据存放位置
   		last_run_metadata_path => "D:/edgeDownload/logstash-7.12.1-windows-x86_64/logstash-7.12.1/mysql/last_run.txt"
   		 # 是否清除last_run_metadata_path的记录，需要增量同步时此字段必须为false；
   		clean_run => false
   		 #
   		 # 同步频率(分 时 天 月 年)，默认每分钟同步一次；
   		schedule => "* * * * *"
   		
   		#数据库中增量同步的字段为timestamp（3）毫秒级别的时候必须设置
   		#否则默认的:sql_last_value将不是毫秒级的时间格式
   		sequel_opts => {
               fractional_seconds => true
           }
   	}
       #多表时可按需添加
       #jdbc{ .......}
   }
   
   filter {
   	json {
   		source => "message"
   	}
   	date {
       match => ["message","UNIX_MS"]
       target => "@timestamp"   
     }
   # 因为时区问题需要修正时间
    ruby { 
      code => "event.set('timestamp', event.get('@timestamp').time.localtime + 8*60*60)" 
    }
    ruby {
      code => "event.set('@timestamp',event.get('timestamp'))"
    }
    
    ruby {
           code => "event.set('create_time', event.get('create_time').time.localtime + 8*60*60)" 
       }    
       ruby {
           code => "event.set('last_update_time', event.get('last_update_time').time.localtime + 8*60*60)" 
       }       
    
   	mutate {
   		remove_field => ["timestamp"]
           remove_field => ["@version"]
       }
   }
   output {
   	elasticsearch {
   		hosts => ["ESIP:9201", "ESIP:9202"]
   		index => "test"
           #文档的唯一标识id,建议为数据库中的主键部分
   		document_id => "%{id}"
   	}
       #多表时：
       #if [type] == "***" {elasticsearch {.....}}
       #if [type] == "***" {elasticsearch {.....}}
   	stdout {
   		codec => json_lines
   	}
   }
   ```

6. 配置mysql的管道pipelines.yml,添加写入：

   ```yaml
    - pipeline.id: mysqltest1
      pipeline.workers: 1
      pipeline.batch.size: 1
      path.config: "/edgeDownload/logstash-7.12.1-windows-x86_64/logstash-7.12.1/mysql/jdbc.conf"
   ```

### 5.3 运行结果

![image-20210511170130687](\ES-ljm\image-20210511170130687.png)

如图按计划将同步数据库表的数据，查看对应ES：

![image-20210511170424844](\ES-ljm\image-20210511170424844.png)

