# kubernetes部署elasticsearch+kibana+logstash+pilot+kafka日志收集系统
### 架构介绍
- elasticsearch：日志存储、搜索
- kibana：日志展示
- logstash：将日志索引信息从kafka取出并导入到elasticsearch
- pilot：阿里开发的专门抓取容器服务的日志收集代理。
  https://github.com/AliyunContainerService/log-pilot
- kafka：作为日志收集系统的中间件架构，pilot将日志存入kafka，等待logstash读取。节点比较多的情况下适用，如果架构不大可以忽略
  
### k8s环境部署elk。
1. 本文档的所有脚本可以从github拉取部署文件
   ``` shell 
    git clone https://github.com/38699678/k8s-study.git 
   ```
2. 本文只讲述部署过程，不讲原理。
   1. 部署nfs插件，支持动态创建. 部署之前先配置nfs
   ``` shell
   #kubectl apply -f addone/nfs-client.yml
   ```
   2. 部署elaseticsearch
   ``` shell
   #cd elasticsearch
   #kubectl apply -f .
   ``` 
   3. 部署kibana
   ``` shell
   #cd kibana
   #kubectl apply -f .
   ```
   4. 部署kafka+zookeeper
   ``` shell
   #cd kafka
   #kubectl apply -f .
   ```
   5. 部署pilot
      1. 根据官方镜像，修改参数并重新制作镜像
      ``` shell
      #cd pilot-docker
      #docker build -t tag:version .
      #docker push 
      ```
      2. 部署pilot
      ``` shell
      #cd pilot-kafka
      #kubectl apply -f pilot-kafa.yml
      ```
   6. 部署logstash
   ``` shell
   #cd logstash
   #kubectl apply -f .
   ```
