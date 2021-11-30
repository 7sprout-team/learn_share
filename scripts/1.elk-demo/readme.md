# 一、准备环境

本文档用于kubernetes安装ELK(ElasticSearch+Kibana+Logstash)环境。
- 硬件：2P/500G/8G, 3台(下文以master, worker1, worker2表示)

- 操作系统：Ubuntu 18.04.4 LTS

- 软件：Docker 20.10.7, Vim工具, Kubernetes v1.22.3(集群)

- 网络：有独立公网IP

- 资源文件：

  [es-pv.yaml](https://github.com/7sprout/learn_share/scripts/1.elk-demo/es-pv.yaml)

  [es-statefulset.yaml](https://github.com/7sprout/learn_share/scripts/1.elk-demo/es-statefulset.yaml)

  [kibana.yaml](https://github.com/7sprout/learn_share/scripts/1.elk-demo/kibana.yaml)

  [logstash.conf](https://github.com/7sprout/learn_share/scripts/1.elk-demo/logstash.conf)

  [logstash-pv.yaml](https://github.com/7sprout/learn_share/scripts/1.elk-demo/logstash-pv.yaml)

  [logstash.yaml](https://github.com/7sprout/learn_share/scripts/1.elk-demo/logstash.yaml)

# 二、安装ElasticSearch

使用kubernetes的部署ElasticSearch集群，数量为3.

~~~ shell
# 在每台服务器上都创建3个文件夹，用于绑定持久化目录
$ mkdir -p /media/k8s-data/es-pv0  /media/k8s-data/es-pv1  /media/k8s-data/es-pv2
$ ls /media/k8s-data
es-pv0  es-pv1  es-pv2
# 接下来的命令只在k8s master节点上执行
# 资源清单文件
# es-pv.yaml：创建持久化存储PersistentVolume
# es-statefulset.yaml：部署ElasticSearch
$ ls 
es-pv.yaml es-statefulset.yaml
# 创建持久化存储PersistentVolume
$ kubectl apply -f es-pv.yaml
# 部署ElasticSearch
$ kubectl apply -f es-statefulset.yaml
# 查看访问端口(30145是外部访问接口)
$ kubectl get svc -n logging
NAME            TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                       
elasticsearch   NodePort   10.108.104.189   <none>        9200:30145/TCP,9300:31095/TCP 
~~~
# 三、安装Kibana

使用kubernetes的部署Kibana.

~~~shell
# 资源清单文件
$ ls
kibana.yaml
# 部署kibana
$ kubectl apply -f kibana.yaml
# 查看端口(浏览器访问http://ip:31411/)
$ kubectl get svc -n logging
NAME            TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                         
elasticsearch   NodePort   10.108.104.189   <none>        9200:30145/TCP,9300:31095/TCP  
kibana          NodePort   10.102.184.191   <none>        5601:31411/TCP                  
~~~
# 四、安装Logstash

~~~shell
# logstash.conf：指定Logstash数据输入、输出插件
# 将文件放入持久化目录，
$ cat logstash.conf
input {
        http {
        }
}

filter {
        json {
                source => "message"
        }
}

output {
        elasticsearch {
                hosts => ["elasticsearch:9200"]
        }
}
~~~

~~~shell
# 在每台服务器上都创建1个文件夹，用于绑定持久化目录
$ mkdir -p /media/k8s-data/logstash-conf
$ ls /media/k8s-data/
es-pv0  es-pv1  es-pv2  logstash-conf 
# 将logstash.conf文件放入持久化目录
$ ls /media/k8s-data/logstash-conf
logstash.conf
# 以下操作是在master节点上
# 资源清单文件: 
# logstash-pv.yaml: 创建持久化存储PersistentVolume
# logstash.yaml：用于部署logstash
$ ls
logstash-pv.yaml  logstash.yaml
# 创建持久卷pv
$ kubectl apply -f logstash-pv.yaml
# 部署logstash
$ kubectl apply -f logstash.yaml
# 查看端口(用http://ip:31047向logstash发送数据)
$ kubectl get svc -n logging
NAME            TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)       
elasticsearch   NodePort   10.108.104.189   <none>        9200:30145/TCP,9300:31095/TCP   
kibana          NodePort   10.102.184.191   <none>        5601:31411/TCP    
logstash        NodePort   10.99.100.229    <none>        8080:31047/TCP
~~~
# 五、案例
外部用http方式向传入数据，在Kibana端查看数据。

## 1、数据传送

数据传送过程：

~~~mermaid
graph TB
应用通过http发送数据--通过logstash-input-http插件写入数据-->Logstash--通过logstash-output-elasticsearch插件写入数据-->ElasticSearch--配置kibana中的es访问地址elasticsearch.url,通过web页面读取es数据-->Kibana分析展示
~~~

~~~shell
# 发送数据
$ curl -XPOST 'http://152.136.178.202:32124' -H 'Content-Type: application/json' -d '{ "Say" : "Hello world!" }'
~~~

## 2、Kibana上查看数据

在Kibana创建Index patterns: logstash*

可以浏览器上访问http://152.136.178.202:31411/点击Discover可以看见上传的数据。

## 3、ES API查看数据

~~~shell
$ curl -XGET "http://152.136.178.202:30145/logstash*/_search" -H 'Content-Type: application/json' | python -m json.tool
...
"hits": [{
			"_index": "logstash-2021.11.30-000001",
			"_type": "_doc",
			"_id": "2N_Mbn0BFYyxVnk--9yD",
			"_score": 1.0,
			"_source": {
				"Say": "Hello world!",
				"@timestamp": "2021-11-30T03:04:27.136Z",
				"@version": "1",
				"headers": {
					"request_method": "POST",
					"http_host": "152.136.178.202:31047",
					"content_length": "26",
					"request_path": "/",
					"http_accept": "*/*",
					"content_type": "application/json",
					"http_user_agent": "curl/7.58.0",
					"http_version": "HTTP/1.1"
				},
				"host": "10.244.0.0"
			}
		}]
...
~~~


