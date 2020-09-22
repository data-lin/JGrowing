# 日志收集监控平台ELK的搭建 #

## 前言 ##

在系统开发过程中，一般都会利用多台服务器做集群部署，保证系统的高可用。如何能够把分散在各个服务器中的日志归集起来做分析处理，是我们需要考虑的一个因素。通常在量小的情况下，我们可以直接使用grep等命令定位日志，在量大的时候就不太适合了。有没有解决方案呢？那就是我们本文要介绍的ELK了。下面我们会讲述日志收集监控平台ELK的背景和搭建演示，最后引入Spring Boot工程进行集成。

## 1、ELK解决了什么问题 ##

### 小系统或单机系统的日志搜索现状？ ###

我们可以直接在日志文件中使用 tail、grep 等命令就可以获得自己想要的信息。

### 大系统或分布式系统的日志面临的问题？ ###

一般大型系统是一个分布式部署的架构，不同的服务模块部署在不同的服务器上。问题出现时，大部分情况需要根据问题暴露的关键信息，定位到具体的服务器和服务模块。一般是采用grep，awk，效率非常低下，如果有一套集中式日志系统，可以提高定位问题的效率。

大规模的分布式面临问题如下：

1. 集群环境下的日志如何查询？
1. 日志量太大如何归档？
1. 文本搜索太慢怎么办？
1. 如何多维度查询？

### 分布式系统的日志搜索的解决方案 ###

需要集中化的日志管理，所有服务器上的日志收集汇总。常见解决思路是建立集中式日志收集系统，将所有节点上的日志统一收集，管理，访问。

一个完整的集中式日志系统，需要包含以下几个主要特点：

- 收集－能够采集多种来源的日志数据
- 传输－能够稳定的把日志数据传输到中央系统
- 存储－支持持久化保存日志数据
- 分析－对数据进行多元化呈现和多维度检索
- 警告－提供监控告警和错误报告

### ELK介绍 ###

ELK是三个开源软件的缩写，分别表示：Elasticsearch , Logstash, Kibana , 是目前主流的一种日志系统。它提供了一整套解决方案，并且都是开源软件，之间互相配合使用，完美衔接，高效的满足了日志的统一收集、管理和访问。

- Elasticsearch是个开源分布式搜索引擎，提供搜集、分析、存储数据三大功能。它的特点有：分布式，零配置，自动发现，索引自动分片，索引副本机制，RESTful风格接口，多数据源，自动搜索负载等。
- Logstash 主要是用来搜集、分析、过滤日志的工具，支持海量的数据获取（支持以 TCP/UDP/HTTP 多种方式收集数据）。一般工作方式为c/s架构，client端安装在需要收集日志的主机上，server端负责将收到的各节点日志进行过滤、修改等操作,再一并 发往Elasticsearch上去。
- Kibana 也是一个开源和免费的工具，可以为 Logstash 和 Elasticsearch 提供的日志分析友好的 Web 界面，帮助汇总、分析和搜索重要数据日志。

### ELK分别扮演什么角色 ###

![](http://qgzyfhov6.hn-bkt.clouddn.com/1.jpg)

- Logstash：负责采集日志

- Elasticsearch：负责存储最终数据、建立索引、提供搜索功能

- Kibana：负责提供可视化界面

## 2、Elasticsearch搭建演示 ##

下面我们将演示如何在一台Linux服务器上安装Elasticsearch（本次演示基于官网最新版本7.9.1）。

一、查看JDK版本

    使用java -version可以查看当前JDK版本，Elasticsearch要求的JDK版本最低是1.8

![](http://qgzyfhov6.hn-bkt.clouddn.com/jdk.jpg)

二、 创建用户

从5.0开始，ElasticSearch 安全级别提高了，不允许采用root帐号启动，这里我们新建一个用户data

	1.创建elk用户组
	  groupadd elk
	2.创建用户data
	  useradd data
	  passwd data
	3.将data用户添加到elk组
	  usermod -G elk data
	4.设置sudo权限
	  visudo
		找到root ALL=(ALL) ALL一行，添加data用户，如下。
		## Allow root to run any commands anywhere
		root ALL=(ALL)   ALL
		data ALL=(ALL)   ALL

![](http://qgzyfhov6.hn-bkt.clouddn.com/data.jpg)

三、安装Elasticsearch

    1.切换到data用户
      su data
    2.直接使用wget线上安装（或者访问官网https://www.elastic.co/cn/downloads，下载到本地，然后执行解压）
      wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.9.1-linux-x86_64.tar.gz
      tar -zxvf  elasticsearch-7.9.1-linux-x86_64.tar.gz
    3.修改目录权限
	  mv elasticsearch-7.9.1/ elasticsearch
	  sudo chown -R data:elk ./
    4.修改ElasticSearch配置elasticsearch.yml
      cd elasticsearch/
      vi config/elasticsearch.yml
	    node.name: node-1
		bootstrap.memory_lock: false
		bootstrap.system_call_filter: false
        network.host: 0.0.0.0
        http.port: 9200
		cluster.initial_master_nodes: ["node-1"]
	5.切换root用户，修改/etc/sysctl.conf
	  su - root
	  vi /etc/sysctl.conf		
	  添加内容如下：
	    vm.max_map_count=262144	
	  查看修改后的sysctl.conf文件			
	  sysctl -p 							
	6.root用户下，修改文件/etc/security/limits.conf
	  vi /etc/security/limits.conf
	  添加如下内容
	    * soft nofile 100001
	    * hard nofile 100002
	    * soft nofile 100001
	    * hard nofile 100002

四、启动

    使用data用户，进入bin目录，执行以下命令（-d表示后台启动）
    ./elasticsearch -d

![](http://qgzyfhov6.hn-bkt.clouddn.com/es.jpg)

五、测试是否安装成功

访问服务器的9200端口，若能够正常返回Elasticsearch的配置信息，表示安装已经完成。

![](http://qgzyfhov6.hn-bkt.clouddn.com/escurl.png)

我们创建一个索引applog，为后面的章节做准备。

    创建索引
    curl -XPUT http://122.51.22.248:9200/applog
    删除索引
    curl -XDELETE http://122.51.22.248:9200/applog
    查看集群节点列表
    curl 'http://122.51.22.248:9200/_cat/nodes?v'
    查看所有的索引
    curl 'http://122.51.22.248:9200/_cat/indices?v'
       
## 3、kibana搭建演示 ##

一、安装kibana

    1.使用data用户，直接wget线上安装（或者访问官网https://www.elastic.co/cn/downloads，下载到本地，然后执行解压）
    wget https://artifacts.elastic.co/downloads/kibana/kibana-7.9.1-linux-x86_64.tar.gz
    tar zxvf kibana-7.9.1-linux-x86_64.tar.gz
    mv kibana-7.9.1-linux-x86_64/ kibana
    2.修改配置
    vim config/kibana.yml 
    把以下注释放开，使配置起作用
    server.port: 5601
    server.host: “0.0.0.0”
    elasticsearch.hosts: ["http://122.51.22.248:9200"]
    kibana.index: “.kibana”

二、启动

    1.直接启动命令
    ./bin/kibana
    2.后台启动命令
    nohup ./bin/kibana &

三、测试是否安装成功

浏览器访问http://122.51.22.248:5601/app/kibana，显示如下即为启动成功。

![](http://qgzyfhov6.hn-bkt.clouddn.com/kibana.jpg)

找到左边导航的Management->Stack Management->Kibana->index patterns,点击Create index pattern创建索引对应的视图。

![](http://qgzyfhov6.hn-bkt.clouddn.com/kibana-createindex.jpg)

输入applog*后依次点击Next step，Create index pattern，即可创建applog对应的索引视图。

![](http://qgzyfhov6.hn-bkt.clouddn.com/applog2.jpg)

导航栏点击kibana->Discover，可以进入applog对应的日志查询台。因为我们applog索引目前还没有传入数据，所以界面显示为空，在后续Spring Boot与ELK的集成演示中我们会通过应用把日志传输到applog索引，到时候就可以看到数据了。

![](http://qgzyfhov6.hn-bkt.clouddn.com/applog.jpg)

## 4、Logstash搭建演示 ##

一、安装Logstash

    直接wget线上安装（或者访问官网https://www.elastic.co/cn/downloads，下载到本地，然后执行解压）
    wget https://artifacts.elastic.co/downloads/logstash/logstash-7.9.1.tar.gz
    tar zxvf logstash-7.9.1.tar.gz
    mv logstash-7.9.1/ logstash

二、新增配置文件log_to_es.conf
	
	vim config/log_to_es.conf	
    在 logstash的主目录下，通过tcp从应用服务器中将日志打入122.51.22.248:9250端口，然后logstash会转发到es的端口9200
	input {
	  tcp {
 	  mode => "server"
 	  port => 9250
 	 }
	}
	output {
 	 elasticsearch {
 	   hosts => ["http://122.51.22.248:9200"]
 	   index => "applog"
 	 }
	}

三、启动

    1.直接启动命令
    ./bin/logstash -f config/log_to_es.conf 
    2.后台启动命令
    ./bin/logstash -f config/log_to_es.conf &

没有报错即表示Logstash已经启动成功，下一节我们会介绍如何引入Spring Boot工程进行集成。

## 5、Spring Boot与ELK的集成演示 ##

一、先说明一下我们的整体架构，日志通过应用服务器传输到Logstash的9250端口，Logstash会将其转发到ElasticSearch的9200端口，最后通过kibana的可视化界面进行查看。

![](http://qgzyfhov6.hn-bkt.clouddn.com/elk.png)

二、我们新建一个Spring Boot的工程elk-consumer，pom文件引入以下注解：

        <dependency>
            <groupId>net.logstash.logback</groupId>
            <artifactId>logstash-logback-encoder</artifactId>
            <version>6.4</version>
        </dependency>

在resources目录下，我们新增日志配置文件logback.xml，通过以下代码指定日志的输出端口：

		<destination>122.51.22.248:9250</destination>

新建一个Controller类ELKController，具体代码如下：

	@RestController
	public class ELKController {
    @RequestMapping(value = "list", produces = "application/json")
    public Map<String, String> listStocks() {
        Map<String, String> result = new HashMap<>();
        result.put("300059", "东方财富");
        result.put("002594", "比亚迪");
        result.put("600030", "中信证券");
        result.put("600036", "招商银行");
        result.put("300026", "红日药业");
        return result;
    	}
	}

启动工程，我们访问list接口，可以看到浏览器能够正常收到返回的数据。

![](http://qgzyfhov6.hn-bkt.clouddn.com/sp.jpg)

访问日志平台http://122.51.22.248:5601/app/kibana，点击discover，找到我们前面创建的applog索引，可以看到日志数据已经能够正常刷新。

![](http://qgzyfhov6.hn-bkt.clouddn.com/ki1.jpg)

我们在Serach中输入关键字东方财富，可以看到跟关键字相关的所有日志都已经被检索出来了。

![](http://qgzyfhov6.hn-bkt.clouddn.com/ki2.jpg)

至此，我们Spring Boot工程和ELK的集成已经完成，具体的工程代码可以访问https://github.com/data-lin/elk-consumer进行下载。

## 6、总结 ##

通过以上篇幅的介绍，我们已经可以搭建属于自己的ELK日志平台了，关于多应用集群环境下的的ELK搭建与配置，课后大家可以自己去实践和摸索，对日志平台感兴趣的小伙伴也可以关注elastic官网，持续追踪ELK产品的最新动态。