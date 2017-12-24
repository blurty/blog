## logstash配置学习（一）

#### 背景
> Logstash is an open source, server-side data processing pipeline that ingests data from a multitude of sources simultaneously, transforms it, and then sends it to your favorite “stash.” 

[logstash](https://www.elastic.co/products/logstash)可以用来Collect（采集）、Parse（解析）、Transform（修改）日志。 

<!--break-->

#### 任务
简单配置logstash。

---

##### 步骤
1. 下载logstash
2. 进入logstash目录。
3. 启动logstash，bin/logstash -f your.cfg

##### your.cfg
    input {
    }

    filter {
    }

    output {
    }

配置文件氛围三大块，input、filter、output。input是输入源，output是输出源，filter是处理日志插件。logstash提供了众多input和output插件，包括udp、http、tcp，还有消息中间件rabbitmq，数据库mongodb、redis等。这里主要说一下输出到udp和http的设置。首先是udp。

### udp

    input {
        stdin { add_field => { "type" => "test" }}
        file {
            path => ["/home/steve/logstash.log"]
            start_position => "beginning"
  	     }
    }

    filter {
    }

    output {
        stdout { codec => rubydebug }
        udp {
    		host => "127.0.0.1"
    		port => 8080
        }
    }

input中我们设置了两个输入源，标准输入和文件输入。

- add_field表示在输入字段中添加一个type字段，它的值为test。
- path设置日志文件的路径。
- `start_position`设置开始监控的位置。有两个值end和beginning。beginning方式表示Logstash启动后，会在系统中记录一个隐藏文件，记录处理过的行号，  当进行挂掉，重新启动后，根据该行号记录续读。  所以start_position只会生效一次。

output中我们设置了两个输出源，标准输出和输出到udp。这个设置简直，一目了然，就不需说了。看一下效果。

    {
          "path" => "/home/steve/logstash.log",
        "@timestamp" => 2017-06-02T09:58:37.199Z,
      "@version" => "1",
          "host" => "ubuntu",
       "message" => "2017/06/02 17:01:24 \t571bfecd-8971-43bb-8d50-d7f23c72db90\t4\t960\t20001\t192.168.1.1\t23753\t127.0.0.1\t9100\t6\twww.baidu.com\twww.loclhost.com\thttp://www.baidu.com/index.html\thttp://www.localhost.com/abc.html\t192.168.1.1:23753:www.baidu.com:http://www.baidu.com/index.html"
    }
    
### http

    input {
	   # stdin { add_field => { "type" => "test" }}
        file {
    	   path => ["/home/steve/logstash.log"]
    	   start_position => "end"
        }
    }
    
    filter {
	   grok {
            #match => ["message", "%{MYDATE:day}"]
            match => ["message", "%{MYDATE:day} %{TIME:time} %{GREEDYDATA:mymessage}"]
	   }
	   mutate {
		  gsub => ["day", "/", "-"]
	   }
    }
    
    output {
        stdout { codec => rubydebug }
	   http {
		  http_method => "post"
		  url => "http://127.0.0.1:18888/test"
		  content_type => "application/octet-stream"
		  #message => "%{time}	%{message}"
		  message => "%{day}	%{day} %{time}%{mymessage}"
		  format => "message"
		  headers => ["X-My-Header", "%{host}", "X-Test", "test"]
	   }
    }

这个稍微复杂一些。首先还是先设置输入源。与以上udp一样。

output中我们设置http。我们采用二进制流的方式传输数据。如果设置了format为message，就必须设置message字段和content-Type字段。我们也可以给http头添加一些自己的信息。如headers所示，数组中两个一对组成key:value。

假如日志中的时间格式为'2017/06/02 17:01:24',但是在我们存入数据库的时候想把日作为一个key，并且数据库只接收2017-06-02的格式。我们就要用到filter来进行设置。

完整的数据为：

    2017/06/02 17:01:24 \t571bfecd-8971-43bb-8d50-d7f23c72db90

首先要用grok正则表达式来对日志分割匹配。这里我们在`./vendor/bundle/jruby/1.9/gems/logstash-patterns-core-4.0.2/patterns/grok-patterns`添加一个自己的pattern。

    MYYEAR (?>\d\d\d\d)
    MYDATE %{MYYEAR}/%{MONTHNUM}/%{MONTHDAY}

最终的输出数据是：

    2017-06-02  2017-06-02 17:01:24 571bfecd-8971-43bb-8d50-d7f23c72db90

同时可以看到，在输入的日志中，时间后边有一个空格+Tab。我们在输出中已经把那个空格给取消了，只剩下了Tab。
