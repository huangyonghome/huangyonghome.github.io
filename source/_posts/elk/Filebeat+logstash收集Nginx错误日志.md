---
title:  Filebeat+logstash收集Nginx错误日志
date: 2020-08-25 22:59:58
tags:  elk
categories: elk
comments: true
copyright: true
---



## Filebeat+logstash收集Nginx错误日志

### 介绍

在上一个笔记<Filebeat+logstash收集Nginx访问日志>中分享了如何收集logstash的访问日志,这篇笔记主要是记录如何收集nginx的错误日志

Nginx 错误日志是运维人员最常见但又极其容易忽略的日志类型之一。Nginx 错误日志即没有统一明确的分隔符，也没有特别方便的正则模式，但通过 logstash 不同插件的组合，还是可以轻松做到数据处理。

---

<!--more-->

### logstash配置

在/etc/logstash/conf.d目录下编辑配置文件`nginx_error.conf`内容如下

```
input {
    beats {
      port => 5044
      client_inactivity_timeout => 600
    }
}
filter {
    if [fields][type] == "nginx-error" {
       grok {
      match => [
          "message", "(?<time_local>%{YEAR}[./-]%{MONTHNUM}[./-]%{MONTHDAY}[- ]%{TIME}) \[%{LOGLEVEL:log_level}\] %{POSINT:pid}#%{NUMBER}: %{GREEDYDATA:error_message}(?:, client: (?<client>%{IP}|%{HOSTNAME}))(?:, server: %{IPORHOST:server}?)(?:, request: %{QS:request})?(?:, upstream: (?<upstream>\"%{URI}\"|%{QS}))?(?:, host: %{QS:request_host})?(?:, referrer: \"%{URI:referrer}\")?",
        "message", "(?<time_local>%{YEAR}[./-]%{MONTHNUM}[./-]%{MONTHDAY}[- ]%{TIME}) \[%{LOGLEVEL:log_level}\]\s{1,}%{GREEDYDATA:error_message}"
      ]
    }
  date {
    match => [ "timestamp" , "dd/MMM/yyyy:HH:mm:ss Z" ]
  }
    mutate {
        convert => [ "status", "integer" ]
        convert => [ "body_bytes","integer" ]
        remove_field => ["message"]

    }
  ruby {
    code => "event.set('log_day', event.get('@timestamp').time.localtime.strftime('%Y%m%d'))"
  }
        }
}

output {
        elasticsearch {
            hosts => ["172.16.20.107:9200"]
            index => "logstash-%{[fields][project]}-%{[fields][type]}-%{+YYYY.MM.dd}"
            #flush_size => 20000
            #idle_flush_time => 10
            #sniffing => true
            #template_overwrite => true
        }
    }
```

---

### Filebeat配置

```
- type: log

  # Change to true to enable this input configuration.
  enabled: true

  # Paths that should be crawled and fetched. Glob based paths.
  paths:
    - /data/logs/nginx/hsq_openapi_beta.error.log

#fields字段用于打标签和索引,在logstash里判断日志来源
  fields:
     type: nginx-error
     project: hsq
     
---output配置,将日志输出到logstash-----
output.logstash:
  # The Logstash hosts
  hosts: ["172.16.20.107:5044"]
```

