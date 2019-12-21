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
2. 创建NFS动态插件 
   elasticsearch是有状态的服务，所以我们要考虑数据的持久化。这里我们用nfs文件系统作为存储，
   用nfs-client-provisioner动态配置nfs server。
    ``` yml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: nfs-client-provisioner 
    spec:
    replicas: 1
    strategy:
        type: Recreate
    selector:
        matchLabels:
        apps: nfs-client-provisioner
    template:
        metadata:
        labels:
            apps: nfs-client-provisioner
        spec:
        serviceAccount: nfs-client-provisioner
        containers:
        - name: nfs-client-provisioner
            image: quay.io/external_storage/nfs-client-provisioner:v3.1.0-k8s1.11
            imagePullPolicy: IfNotPresent
            volumeMounts:
            - name: nfs-client-root
            mountPath: /persistentvolumes
            env:
            - name: PROVISIONER_NAME
            value: k8s-logs-sc  #此处在后面创建存储类的时候会用到
            - name: NFS_SERVER
            value: 172.18.178.239 #此处配置nfs server地址和路径
            - name: NFS_PATH
            value: /data/logs
        volumes:
        - name: nfs-client-root
            nfs:
            server: 172.18.178.239
            path: /data/logs
    ```
    ``` shell 
    #kubectl apply -f nfs-client.yml
    #kubectl get sc  
    ```
3. 部署elasticsearch   
   1. 创建动态存储类  
   ``` yml
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: es-data-db
   provisioner: k8s-logs-sc #此处就是上面提醒的值
   ```
   2. 创建elastic部署文件  
   ``` yml
    apiVersion: apps/v1 
    kind: StatefulSet
    metadata:
    name: elasticsearch
    labels:
        apps: elasticsearch
    namespace: kube-ops
    spec:
      replicas: 3
      revisionHistoryLimit: 5
      selector:
          matchLabels:
          apps: elasticsearch
      serviceName: elasticsearch
      template:
          metadata:
          labels: 
              apps: elasticsearch
          namespace: kube-ops
          spec:
          containers:
          - name: elasticsearch
              image: elasticsearch:7.4.2
              imagePullPolicy: IfNotPresent
              resources:
              requests:
                  cpu: 100m
              limits:
                  cpu: 1000m
                  memory: 2Gi
              ports:
              - name: rest 
              containerPort: 9200
              - name: inter-node
              containerPort: 9300 
              volumeMounts:
              - name: data
              mountPath: /usr/share/elasticsearch/data
              env:
              - name: cluster.name
              value: panda-logs
              - name: node.name
              valueFrom:
                  fieldRef:
                  fieldPath: metadata.name
              - name: discovery.seed_hosts #此处是配置elastic集群。
              value: "elasticsearch-0.elasticsearch.kube-ops,elasticsearch-1.elasticsearch.kube-ops,elasticsearch-2.elasticsearch.kube-ops"
              - name: cluster.initial_master_nodes
              value: "elasticsearch-0,elasticsearch-1,elasticsearch-2"
              - name: ES_JAVA_OPTS
              value: "-Xms512m -Xmx512m"
          initContainers:
          - name: chown-data
              image: busybox:1.31.1
              imagePullPolicy: IfNotPresent
              command: ["sh","-c","chown -R 1000:1000 /usr/share/elasticsearch/data"]
              securityContext:
                privileged: true
              volumeMounts:
              - name: data
                mountPath: /usr/share/elasticsearch/data
          - name: increase-vm-max-map
              image: busybox
              command: ["sysctl", "-w", "vm.max_map_count=262144"]
              securityContext:
                privileged: true
          - name: increase-fd-ulimit
              image: busybox
              command: ["sh", "-c", "ulimit -n 65536"]
              securityContext:
                privileged: true
      volumeClaimTemplates:  #此处设置存储声明模板
          - metadata:
              name: data
              labels:
                apps: elasticsearch
          spec:
              accessModes: ["ReadWriteOnce"]
              storageClassName: es-data-db #调用上方创建的存储类
              resources:
                requests:
                  storage: 50Gi #申请的存储大小，但是发现这个不是个硬限制
       ```
       
  3. 配置service  
    ``` yml  
    apiVersion: v1 
    kind: Service 
    metadata:
      name: elasticsearch
    namespace: kube-ops
      labels:
        apps: elasticsearch
    spec:
      selector:
        apps: elasticsearch 
      ports:
      - name: rest 
        port: 9200
      - port: 9300
        name: inter-node
      clusterIP: None   
      ```


