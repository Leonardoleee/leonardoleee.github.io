---
layout: post
title:  "图见:ELK:工作原理"
date:   2018-11-30 16:29:21 +0800
tags: Elasticsearch Logstash Kibana
color: rgb(255,90,90)
cover: 'http://47.105.135.189/img/elk/000-elasticsearch.jpg'
subtitle: 'ELK工作原理'
---

### 一图胜千言
 + 基础架构
![avatar](http://47.105.135.189/img/elk/001-ELK架构.png)  
 + 工作原理  
![avatar](http://47.105.135.189/img/elk/002-ELK工作原理.png)
 + Logstash工作原理
 ![avatar](http://47.105.135.189/img/elk/003-Logstash工作原理.png)
  + Logstash工作流程
 ![avatar](http://47.105.135.189/img/elk/004-Logstash工作流程.png)
  + ELK整体部署图
 ![avatar](http://47.105.135.189/img/elk/005-ELK整体部署图.png)

### ELK 安装配置简化过程
```
1 基本配置
    vim /etc/hosts
    192.168.2.61 		master-node
    192.168.2.62       data-node1
    192.168.2.63       data-node2
    wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.0.0.rpm
    rpm -ivh elasticsearch-6.0.0.rpm

    elasticsearch.yml
    jvm.options
    log4j2.properties

    vim /etc/elasticsearch/elasticsearch.yml 
    cluster.name: master-node  # 集群中的名称
    node.name: master  # 该节点名称
    node.master: true  # 意思是该节点为主节点
    node.data: false  # 表示这不是数据节点
    network.host: 0.0.0.0  # 监听全部ip，在实际环境中应设置为一个安全的ip
    http.port: 9200  # es服务的端口号
    discovery.zen.ping.unicast.hosts: ["192.168.2.61", "192.168.2.62", "192.168.2.63"] # 配置自动发现

    scp /etc/elasticsearch/elasticsearch.yml data-node1:/tmp/
    scp /etc/elasticsearch/elasticsearch.yml data-node2:/tmp/

    cp /tmp/elasticsearch.yml /etc/elasticsearch/elasticsearch.yml
    systemctl start elasticsearch.service
    curl '192.168.2.61:9200/_cluster/health?pretty'
    curl '192.168.2.61:9200/_cluster/state?pretty'
2 kibana配置
    wget https://artifacts.elastic.co/downloads/kibana/kibana-6.0.0-x86_64.rpm
    rpm -ivh kibana-6.0.0-x86_64.rpm

    vim /etc/kibana/kibana.yml
    server.port: 5601  # 配置kibana的端口
    server.host: 192.168.2.61  # 配置监听ip
    # 配置es服务器的ip，如果是集群则配置该集群中主节点的ip
    elasticsearch.url: "http://192.168.2.61:9200" 
    # 配置kibana的日志文件路径，不然默认是messages里记录日志 
    logging.dest: /var/log/kibana.log  

    touch /var/log/kibana.log; chmod 777 /var/log/kibana.log
    systemctl start kibana
    http://192.168.2.61:5601/ 
3 logstash配置
    192.168.2.62
    wget https://artifacts.elastic.co/downloads/logstash/logstash-6.0.0.rpm
    rpm -ivh logstash-6.0.0.rpm

    vim /etc/logstash/conf.d/syslog.conf 
    input {  # 定义日志源
                        syslog {
                                            type => "system-syslog"  # 定义类型
                                            port => 10514    # 定义监听端口
                        }
    }
    output {  # 定义日志输出
                        stdout {
                                            codec => rubydebug  # 将日志输出到当前的终端上显示
                        }
    }

    cd /usr/share/logstash/bin
    检查配置
    ./logstash --path.settings /etc/logstash/ -f /etc/logstash/conf.d/syslog.conf --config.test_and_exit

    配置kibana服务器的ip以及配置的监听端口
    vim /etc/rsyslog.conf
    #### RULES ####

    *.* @@192.168.2.62:10514
    systemctl restart rsyslog

    指定配置文件，启动logstash
    cd /usr/share/logstash/bin
    ./logstash --path.settings /etc/logstash/ -f /etc/logstash/conf.d/syslog.conf
    
logstash收集nginx日志
vim /etc/logstash/conf.d/nginx.conf 

input {
  file {  # 指定一个文件作为输入源
    path => "/var/log/nginx/access.log"  # 指定文件的路径
    start_position => "beginning"  # 指定何时开始收集
    type => "nginx"  # 定义日志类型，可自定义
  }
}
filter {  # 配置过滤器
    grok {
        match => { "message" => "%{IPORHOST:http_host} %{IPORHOST:clientip} - %{USERNAME:remote_user} \[%{HTTPDATE:timestamp}\] \"(?:%{WORD:http_verb} %{NOTSPACE:http_request}(?: HTTP/%{NUMBER:http_version})?|%{DATA:raw_http_request})\" %{NUMBER:response} (?:%{NUMBER:bytes_read}|-) %{QS:referrer} %{QS:agent} %{QS:xforwardedfor} %{NUMBER:request_time:float}"}  # 定义日志的输出格式
    }
    geoip {
        source => "clientip"
    }
}
output {
    stdout { codec => rubydebug }
    elasticsearch {
        hosts => ["192.168.2.61:9200"]
        index => "nginx-test-%{+YYYY.MM.dd}"
  }
}

cd /usr/share/logstash/bin
./logstash --path.settings /etc/logstash/ -f /etc/logstash/conf.d/nginx.conf --config.test_and_exit

cd /etc/nginx/http_virtual_host.d
vim elk.conf
server {
      listen 80;
      server_name elk.test.com;

      location / {
          proxy_pass      http://192.168.2.61:5601;
          proxy_set_header Host   $host;
          proxy_set_header X-Real-IP      $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      }

      access_log  /tmp/elk_access.log main2;
}

vim 
log_format main2 '$http_host $remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$upstream_addr" $request_time';
nginx -t
nginx -s reload
配置hosts  192.168.2.62 elk.eichong.com

ls /var/log/nginx/access.log
wc -l !$
重启logstash服务，生成日志的索引
systemctl restart logstash

重启完成后，在es服务器上检查是否有nginx-test开头的索引生成
curl '192.168.2.61:9200/_cat/indices?v'

nginx-test索引已经生成了，那么这时就可以到kibana上配置该索引
managent  index patterns   create index patterns
discover
http://192.168.2.61:5601/status 查看状态

```

### 最新版本yum安装
```
001 elasticsearch

rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
vim /etc/yum.repos.d/elasticsearch.repo

[elasticsearch-6.x]
name=Elasticsearch repository for 6.x packages
baseurl=https://artifacts.elastic.co/packages/6.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md

yum install elasticsearch -y

002 kibana
vim /etc/yum.repos.d/kibana.repo

[kibana-6.x]
name=Kibana repository for 6.x packages
baseurl=https://artifacts.elastic.co/packages/6.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md

yum install kibana -y

003 logstash
vim /etc/yum.repos.d/logstash.repo

[logstash-6.x]
name=Elastic repository for 6.x packages
baseurl=https://artifacts.elastic.co/packages/6.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
```
### 主要配置文件
```
001 elasticsearch
cat /etc/elasticsearch/elasticsearch.yml |grep ^[^#]

cluster.name: my-elk
node.name: master
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: 0.0.0.0
http.port: 9200
discovery.zen.ping.unicast.hosts: ["172.19.1.216", "172.19.1.217"]

002 kibana
cat /etc/kibana/kibana.yml |grep ^[^#]

server.port: 5601
server.host: "172.19.1.216"
elasticsearch.url: "http://172.19.1.216:9200"
logging.dest: /var/log/kibana.log  # 文件需创建并授权

003 logstash
```
### 汉化
```
https://github.com/anbai-inc/Kibana_Hanization
```

### 其他优秀博客
```
https://www.cnblogs.com/kevingrace/p/5919021.html
http://blog.51cto.com/zero01/2079879
```
