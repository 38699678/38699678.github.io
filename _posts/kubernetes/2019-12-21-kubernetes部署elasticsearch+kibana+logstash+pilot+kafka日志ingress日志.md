#### 创建ceph存储
- 在ceph的管理节点创建elk的存储池
  - ceph osd pool create elk-pool-data 64 64 replicated
- 指定该存储池采用哪种形式的存储方式，rbd为块存储类型
  - ceph osd pool application enable  elk-pool-data rbd
  - 
#### 配置存储
- 配置和绑定ceph存储
    - 编写ceph-secrets.yml文件
      - 这个文件是用于绑定ceph的身份认证token的
    ``` bash 
    apiVersion: v1
    kind: Secret
    metadata:
    name: ceph-secret
    namespace: kube-ops
    stringData:
    userID: admin
    #此处为ceph集群中的认证token 在ceph.client.admin.keyring中获取
    userKey: AQDVX19e+hXhFxAAmU7jiLPL6NKB7xxxxxxxx
    ```
    - 创建elk-storageclass.yml.用于创建storageclass
    ``` bash 
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
        name: elk-data-db
        namespace: kube-ops
    provisioner: rbd.csi.ceph.com
    parameters:
        #此id为ceph的clusterid，可以在ceph中执行ceph -s命令获得
        clusterID: "ccc231ee-e41c-4d2d-b126-2bce5xxxxx"
        #此为之前创建的ceph的存储池
        pool: elk-pool-data
        #指定上面创建的ceph认证文件，如果elk指定namespace，需要在这说明
        csi.storage.k8s.io/provisioner-secret-name: ceph-secret
        csi.storage.k8s.io/provisioner-secret-namespace: kube-ops
        csi.storage.k8s.io/node-stage-secret-name: ceph-secret
        csi.storage.k8s.io/node-stage-secret-namespace: kube-ops
    reclaimPolicy: Delete
    mountOptions:
    - discard
    ```
    - 执行以上ceph配置文件
      - kubectl create ns kube-ops  #如果没有创建名称空间，这部就要创建
      - kubectl apply -f ceph-secrets.yml
      - kubectl apply -f elk-storageclass.yml
    - 检查storageclass是否创建成功
      - kubectl get sc

#### 部署elasticsearch
- 部署elasticsearch-deployment.yml
``` bash
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
      #此处需要调节，根据你实际要部署的节点label进行填写
      nodeSelector:
        pro-pub-tools: pro-pub-tools
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
          value: cluster.local
        - name: node.name
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        #此处，如果你要创建3各节点的elastic。那么这块就按这个填写，如果多余3个，则按顺序填写
        - name: discovery.seed_hosts
          value: "elasticsearch-0.elasticsearch.kube-ops,elasticsearch-1.elasticsearch.kube-ops,elasticsearch-2"
        - name: cluster.initial_master_nodes
        #此处对应上一条的配置
          value: "elasticsearch-0,elasticsearch-1,elasticsearch-2"
        - name: ES_JAVA_OPTS
          value: "-Xms512m -Xmx512m"
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
  volumeClaimTemplates:
    - metadata:
        name: data
        labels:
          apps: elasticsearch
      spec:
        accessModes: ["ReadWriteOnce"]
        #此处配置之前创建的storageclass
        storageClassName: elk-data-db
        resources:
          requests:
            storage: 2000Gi
```
- 创建service文件
``` bash
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: kube-ops
  labels:
    apps: elasticsearch
spec:
  type: NodePort
  selector:
    apps: elasticsearch
  ports:
  - name: rest
    port: 9200
    nodePort: 9200
  - port: 9300
    name: inter-node
```
- 部署elasticsearch
  - kubectl apply -f elasticsearch-statefulset.yml
  - kubectl apply -f elasticsearch-svc.yml

#### 部署kibana
- 编写kibana-deploy.yml文件
``` bash 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  labels:
    app: kibana
  namespace: kube-ops
spec:
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
    #配置要部署的主机节点label
      nodeSelector:
        pro-pub-tools: pro-pub-tools
      containers:
      - name: kibana
        image: kibana:7.4.2
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5601
          name: kibana-port
        resources:
          requests:
            cpu: 100m
        env:
        #此处指定elasticsearch的集群地址。elasticsearch是servicename
        - name: ELASTICSEARCH_HOSTS
          value: http://elasticsearch:9200
```
- 编写service文件
``` bash 
apiVersion: v1
kind: Service
metadata:
  name: kibana
  labels:
    app: kibana
  namespace: kube-ops
spec:
  type: NodePort
  ports:
  - port: 5601
  #配置要暴露的端口
    nodePort: 30003
  selector:
    app: kibana
```
- 部署kibana
  - kubectl apply -f kibana-deploy.yml
  - kubectl apply -f kibana-svc.yml

#### 部署logstash
- 编写logstash部署文件
``` bash 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash
  namespace: kube-ops
  labels:
    apps: logstash
spec:
  replicas: 3
  selector:
    matchLabels:
      apps: logstash
  template:
    metadata:
      labels:
        apps: logstash
    spec:
    #配置要部署的节点label
      nodeSelector:
        pro-pub-tools: pro-pub-tools
      containers:
      - name: logstash
        image: logstash:7.5.0
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: conf
          mountPath: /usr/share/logstash/pipeline/
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
      volumes:
      - name: conf
        configMap:
          name: logstash-config
```
- 编写logstash的配置文件
``` bash 
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-config
  labels:
    app: logstash
  namespace: kube-ops
data:
  logstash.conf: |-
  #配置从哪里抓取数据。这里我们用的是kafka，我们就将kafka的所有节点都写入这里
    input {
        kafka {
            bootstrap_servers => ["kafka-0.kafka-headless:9092,kafka-1.kafka-headless:9092,kafka-2.kafka-headless:9092"]
                    topics_pattern => "^[a-zA-Z].*"
                    type => "logs"
                    consumer_threads => 10
                    decorate_events => true
                    codec => "json"
                    auto_offset_reset => "latest"
                    group_id => "logstash"
            }
    }
    #对抓取数据进行操作
    filter {
        #抓取的日志格式为：
        #{"log":"{\"time\": \"2020-06-10T01:03:23+00:00\", \"remote_addr\": \"-\", \"http_x_forwarded_for\": \"8.210.25.214\", 
        #\"proxy_add_x_forwarded_for\": \"8.210.25.214, 10.5.16.1\", \"request_id\": \"1ace50bb7b7fa4fca530b9a79630030b\",\"remote_user\": \"-\",
        # \"bytes_sent\": 237, \"request_time\": 0.001, \"status\":200, \"vhost\": \"api.sportxxxwo8.com\", \"request_proto\": 
        # \"HTTP/1.1\", \"path\": \"/\", \"request_query\": \"-\", \"request_length\": 162, \"duration\": 0.001,\"method\": 
        # \"GET\", \"http_referrer\": \"-\",\"http_user_agent\": \"Go-http-client/1.1\" }\n","stream":"stdout","time":"2020-06-10T01:03:23.353283199Z"}
        #将抓取的数据转换成json格式。log是抓取数据中的一个根节点，他下面的元素是日志的完整记录
        json {
            source => "log"
         }
        #删减要发送到elk的数据中不需要的标签
        mutate {
            remove_field=> ["docker_container","k8s_container_name","stream","host","log","remote_user","topic","type"]
        }
    }
    #发送到
    output {
            elasticsearch {
                    hosts => ["elasticsearch:9200"]
                    #index => ["%{[@metadata][kafka][topic]}-%{+YYYY.MM.dd}"]
                    index => "logstash-nginx-%{+YYYY.MM.dd}"
            }
    }
```
- 部署logstash
  - kubectl apply -f logstash-deploy.yml
  - kubectl apply -f logstash-cm.yml
  
#### 部署kafka
- 这套环境中我们用kafka+zookeeper做中间件，用于存储pilot抓取得日志
- 编写存储部署文件,这里我们调用得也是elk-pool-data这个存储池。
``` bash 
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: kafka-data
   namespace: kube-ops
provisioner: rbd.csi.ceph.com
parameters:
  clusterID: "ccc231ee-e41c-4d2d-b126-2bce5xxxxxxx"
  pool: elk-pool-data
  csi.storage.k8s.io/provisioner-secret-name: ceph-secret
  csi.storage.k8s.io/provisioner-secret-namespace: kube-ops
  csi.storage.k8s.io/node-stage-secret-name: ceph-secret
  csi.storage.k8s.io/node-stage-secret-namespace: kube-ops
reclaimPolicy: Delete
mountOptions:
   - discard
``` 
- 编写zookeeper部署文件
``` bash
apiVersion: v1
kind: Service
metadata:
  name: zk-headless
  labels:
    app: zk-headless
  namespace: kube-ops
spec:
  ports:
  - port: 2888
    name: server
  - port: 3888
    name: leader-election
  - port: 2181
    name: client
  clusterIP: None
  selector:
    app: zk
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: zk-config
  namespace: kube-ops
data:
  ensemble: "zk-0;zk-1;zk-2"
  jvm.heap: "2G"
  tick: "2000"
  init: "10"
  sync: "5"
  client.cnxns: "60"
  snap.retain: "3"
  purge.interval: "1"
---
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: zk-budget
  namespace: kube-ops
spec:
  selector:
    matchLabels:
      app: zk
  minAvailable: 2
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: zk
  namespace: kube-ops
spec:
  selector:
    matchLabels:
      app: zk
  serviceName: zk-headless
  replicas: 3
  template:
    metadata:
      labels:
        app: zk
      annotations:
        pod.alpha.kubernetes.io/initialized: "true"

    spec:
    #配置要部署的节点的label
      nodeSelector:
        pro-pub-tools: pro-pub-tools
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: "app"
                    operator: In
                    values:
                    - zk-headless
              topologyKey: "kubernetes.io/hostname"
      containers:
      - name: k8szk
        imagePullPolicy: Always
        image: gcr.io/google_samples/k8szk:v1
        resources:
          requests:
            memory: "256Mi"
            cpu: "300m"
        ports:
        - containerPort: 2181
          name: client
        - containerPort: 2888
          name: server
        - containerPort: 3888
          name: leader-election
        env:
        - name : ZK_ENSEMBLE
          valueFrom:
            configMapKeyRef:
              name: zk-config
              key: ensemble
        - name : ZK_HEAP_SIZE
          valueFrom:
            configMapKeyRef:
                name: zk-config
                key: jvm.heap
        - name : ZK_TICK_TIME
          valueFrom:
            configMapKeyRef:
                name: zk-config
                key: tick
        - name : ZK_INIT_LIMIT
          valueFrom:
            configMapKeyRef:
                name: zk-config
                key: init
        - name : ZK_SYNC_LIMIT
          valueFrom:
            configMapKeyRef:
                name: zk-config
                key: tick
        - name : ZK_MAX_CLIENT_CNXNS
          valueFrom:
            configMapKeyRef:
                name: zk-config
                key: client.cnxns
        - name: ZK_SNAP_RETAIN_COUNT
          valueFrom:
            configMapKeyRef:
                name: zk-config
                key: snap.retain
        - name: ZK_PURGE_INTERVAL
          valueFrom:
            configMapKeyRef:
                name: zk-config
                key: purge.interval
        - name: ZK_CLIENT_PORT
          value: "2181"
        - name: ZK_SERVER_PORT
          value: "2888"
        - name: ZK_ELECTION_PORT
          value: "3888"
        command:
        - sh
        - -c
        - zkGenConfig.sh && zkServer.sh start-foreground
        readinessProbe:
          exec:
            command:
            - "zkOk.sh"
          initialDelaySeconds: 15
          timeoutSeconds: 5
        livenessProbe:
          exec:
            command:
            - "zkOk.sh"
          initialDelaySeconds: 15
          timeoutSeconds: 5
        volumeMounts:
        - name: datadir
          mountPath: /var/lib/zookeeper
      securityContext:
        runAsUser: 1000
        fsGroup: 1000
  volumeClaimTemplates:
  - metadata:
      name: datadir
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 50Gi
    #指定要绑定的存储类
      storageClassName: kafka-data
```
- 编写kafka部署文档
``` bash
apiVersion: v1
kind: Service
metadata:
  name: kafka-headless
  namespace: kube-ops
spec:
  ports:
  - name: broker
    port: 9092
    protocol: TCP
    targetPort: 9092
  selector:
    app: kafka
  sessionAffinity: None
  type: ClusterIP
  clusterIP: None

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app: kafka
  name: kafka
  namespace: kube-ops
spec:
  podManagementPolicy: OrderedReady
  replicas: 3
  revisionHistoryLimit: 1
  selector:
    matchLabels:
      app: kafka
  serviceName: kafka-headless
  template:
    metadata:
      labels:
        app: kafka
    spec:
    #配置要部署的节点label
      nodeSelector:
        pro-pub-tools: pro-pub-tools
      containers:
      - command:
        - sh
        - -exc
        - |
          unset KAFKA_PORT && \
          export KAFKA_BROKER_ID=${HOSTNAME##*-} && \
          export KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://${POD_IP}:9092 && \
          exec /etc/confluent/docker/run
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: status.podIP
        - name: KAFKA_HEAP_OPTS
          value: -Xmx1G -Xms1G
        - name: KAFKA_ZOOKEEPER_CONNECT
        #设置zookeeper的pod地址
          value: zk-0.zk-headless:2181,zk-1.zk-headless:2181,zk-2.zk-headless:2181
        - name: KAFKA_LOG_DIRS
          value: /opt/kafka/data/logs
        - name: KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR
          value: "3"
        - name: KAFKA_JMX_PORT
          value: "9999"
        image: confluentinc/cp-kafka:3.3.0
        imagePullPolicy: IfNotPresent
        livenessProbe:
          exec:
            command:
            - sh
            - -ec
            - /usr/bin/jps | /bin/grep -q SupportedKafka
          failureThreshold: 3
          initialDelaySeconds: 30
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 5
        name: kafka-broker
        ports:
        - containerPort: 9092
          name: kafka
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          initialDelaySeconds: 30
          periodSeconds: 10
          successThreshold: 1
          tcpSocket:
            port: kafka
          timeoutSeconds: 5
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /opt/kafka/data
          name: datadir
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 60
  updateStrategy:
    type: OnDelete
  volumeClaimTemplates:
  - metadata:
      name: datadir
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 100Gi
    #配置调用的存储类
      storageClassName: kafka-data
```

#### 部署log-pilot
- 这里我们用的是在flunted基础上做的pilot.他的类型是daemonset格式，如果不做限制，会在每个节点上创建一个收集pod
``` bash 
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-pilot
  namespace: kube-ops
  labels:
    k8s-app: log-pilot
spec:
  updateStrategy:
    type: RollingUpdate
  selector:
    matchLabels:
      k8s-app: log-pilot
  template:
    metadata:
      labels:
        k8s-app: log-pilot
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
    #配置要部署的节点label
      nodeSelector:
        ingress: nginx-ingress
      imagePullSecrets:
      - name: regsecret
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: log-pilot
      #此处可以用他的官方镜像
        image: 10.5.19.239/idc/pilot-fluentd:v4
        imagePullPolicy: Always
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        env:
        #设置kafka的地址
          - name: FLUENTD_OUTPUT
            value: "kafka"
          - name: "KAFKA_BROKERS"
            value: "kafka-0.kafka-headless:9092,kafka-1.kafka-headless:9092,kafka-2.kafka-headless:9092"
        #此处如果数据不是json格式，会转成json格式存放
          - name: "KAFKA_OUTPUT_DATA_TYPE"
            value: "json"
          - name: "FLUENTD_LOG_LEVEL"
            value: "error"
          - name: "NODE_NAME"
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
        volumeMounts:
        - name: sock
          mountPath: /var/run/docker.sock
        - name: root
          mountPath: /host
          readOnly: true
        - name: localtime
          mountPath: /etc/localtime
        - name: logs
          mountPath: /opt/logs
        securityContext:
          capabilities:
            add:
            - SYS_ADMIN
      terminationGracePeriodSeconds: 30
      volumes:
      - name: sock
        hostPath:
          path: /var/run/docker.sock
      - name: root
        hostPath:
          path: /
      - name: localtime
        hostPath:
          path: /etc/localtime
      - name: logs
        hostPath:
          path: /opt/logs
```
- 部署pilot
  - kubectl apply -f pilot-kafka.yml
  
#### 配置nginx-ingress。加入日志收集功能，修改日志格式
- 此文档中nginx-ingress采用helm部署。
- 配置values.yml
  - 修改control的config选项
  ``` bash
  config:
    log-format-upstream: '{"time": "$time_iso8601", "remote_addr": "$proxy_protocol_addr", "http_x_forwarded_for": "$http_x_forwarded_for", "proxy_add_x_forwarded_for": "$proxy_add_x_forwarded_for", "request_id": "$req_id","remote_user": "$remote_user", "bytes_sent": $bytes_sent, "request_time": $request_time, "status":$status, "vhost": "$host", "request_proto": "$server_protocol", "path": "$uri", "request_query": "$args", "request_length": $request_length, "duration": $request_time,"method": "$request_method", "http_referrer": "$http_referer","http_user_agent": "$http_user_agent" }'
  ```
  - 修改环境变量选项
  ``` bash
  extraEnvs:
    - name: aliyun_logs_nginx-ingress      value: "stdout"    - name: aliyun_logs_nginx-ingress_format
      value: "json"
  ```
- 部署nginx-ingress
  - helm install nginx-ingess -n kube-system .
