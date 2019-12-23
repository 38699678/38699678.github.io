### 一.部署metrics server
k8s实现hap自动扩缩容，必须有metrics server的支持。所以要现在k8s集群中部署metrics-server服务。  
- 编辑权限配置文件。metrics-server-rbac.yml
``` yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:metrics-server
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - nodes
  - nodes/stats
  - namespaces
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - "apps"
  resources:
  - deployments
  verbs:
  - get
  - list
  - update
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:metrics-server
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:metrics-server
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: metrics-server-auth-reader
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: extension-apiserver-authentication-reader
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
 
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: metrics-server:system:auth-delegator
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
 
```
- 编辑Apiservice文件。metrics-server-apiservice.yml
``` yml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1beta1.metrics.k8s.io
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  service:
    name: metrics-server
    namespace: kube-system
  group: metrics.k8s.io
  version: v1beta1
  insecureSkipTLSVerify: true
  groupPriorityMinimum: 100
  versionPriority: 100  
```
- 编辑服务配置文件，metrics-server-svc.yml
``` yml
apiVersion: v1
kind: Service
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "Metrics-server"
spec:
  selector:
    k8s-app: metrics-server
  ports:
  - port: 443
    protocol: TCP
    targetPort: https
```
- 编辑部署配置文件，metrics-server-deploy.yml   
特别注意，如果你是从kubernetes的github上下载的部署文件，里面会有几个坑，我会在配置文件中指出。如果你再配置中遇到以下报错。请参考配置文件做修改  
``` error
kubectl logs -f metric-metrics-server-5f9f586954-bspvm -c metrics-server -n kube-system
E1212 02:44:32.223649       1 manager.go:111] unable to fully collect metrics: [unable to fully scrape metrics from source kubelet_summary:node3: unable to fetch metrics from Kubelet node3 (node3): Get https://node3:10250/stats/summary?only_cpu_and_memory=true: dial tcp: lookup node3 on 10.96.0.10:53: no such host, unable to fully scrape metrics from source kubelet_summary:node2: unable to fetch metrics from Kubelet node2 (node2): Get https://node2:10250/stats/summary?only_cpu_and_memory=true: dial tcp: lookup node2 on 10.96.0.10:53: no such host, unable to fully scrape metrics from source kubelet_summary:node4: unable to fetch metrics from Kubelet node4 (node4): Get https://node4:10250/stats/summary?only_cpu_and_memory=true: dial tcp: lookup node4 on 10.96.0.10:53: no such host, unable to fully scrape metrics from source kubelet_summary:node1: unable to fetch metrics from Kubelet node1 (node1): Get https://node1:10250/stats/summary?only_cpu_and_memory=true: dial tcp: lookup node1 on 10.96.0.10:53: no such host, unable to fully scrape metrics from source kubelet_summary:master: unable to fetch metrics from Kubelet master (master): Get https://master:10250/stats/summary?only_cpu_and_memory=true: dial tcp: lookup master on 10.96.0.10:53: no such host]
```
``` yml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: metrics-server-config
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: EnsureExists
data:
  NannyConfiguration: |-
    apiVersion: nannyconfig/v1alpha1
    kind: NannyConfiguration
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-server-v0.3.6
  namespace: kube-system
  labels:
    k8s-app: metrics-server
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    version: v0.3.6
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
      version: v0.3.6
  template:
    metadata:
      name: metrics-server
      labels:
        k8s-app: metrics-server
        version: v0.3.6
      annotations:
        seccomp.security.alpha.kubernetes.io/pod: 'docker/default'
    spec:
      priorityClassName: system-cluster-critical
      serviceAccountName: metrics-server
      nodeSelector:
        kubernetes.io/os: linux
      containers:
      - name: metrics-server
        image: k8s.gcr.io/metrics-server-amd64:v0.3.6
        command:
        - /metrics-server
        - --metric-resolution=30s
        - --kubelet-port=10250  #此处需要注释掉或者修改成10250，kubelet的端口现在是10250.
        - --kubelet-insecure-tls # metrics-server 会通过 kubelet 的 10250 端口获取信息，使用的是 hostname，我们部署集群的时候在节点的 /etc/hosts 里面添加了节点的 hostname 和 ip 的映射，但是是我们的 metrics-server 的 Pod 内部并没有这个 hosts 信息，当然也就不识别 hostname 了
        - --kubelet-preferred-address-types=InternalIP
        ports:
        - containerPort: 443
          name: https
          protocol: TCP
      - name: metrics-server-nanny
        image: k8s.gcr.io/addon-resizer:1.8.7
        resources:
          limits:
            cpu: 100m
            memory: 300Mi
          requests:
            cpu: 5m
            memory: 50Mi
        env:
          - name: MY_POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: MY_POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
        volumeMounts:
        - name: metrics-server-config-volume
          mountPath: /etc/config
        command:
          - /pod_nanny
          - --config-dir=/etc/config
          #- --cpu={{ base_metrics_server_cpu }} #以下这些地方需要自己手动指定，一开始我还以为是会动态获取，发现我错了。如果不指定就注释。
          - --extra-cpu=0.5m
          #- --memory={{ base_metrics_server_memory }}
          #- --extra-memory={{ metrics_server_memory_per_node }}Mi
          - --threshold=5
          - --deployment=metrics-server-v0.3.6
          - --container=metrics-server
          - --poll-period=300000
          - --estimator=exponential
          # Specifies the smallest cluster (defined in number of nodes)
          # resources will be scaled to.
          #- --minClusterSize={{ metrics_server_min_cluster_size }}
          - --minClusterSize=4  #此处添加参与扩展的node节点数
      volumes:
        - name: metrics-server-config-volume
          configMap:
            name: metrics-server-config
      tolerations:
        - key: "CriticalAddonsOnly"
          operator: "Exists"
```
- 执行#kubectl apply -f .  
- 查看日志输出是否有报错导致无法启动。  
``` shell

#kubectl logs  metrics-server-v0.3.6-6fcb7df8d4-hwhwr -c metrics-server -n kube-system
#ubectl logs  metrics-server-v0.3.6-6fcb7df8d4-hwhwr -c metrics-server-nanny -n kube-system
[root@master hpa]# kubectl top nodes
NAME     CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%  
master   90m          4%     962Mi           35%      
node1    1159m        57%    1803Mi          67%      
node2    51m          2%     1573Mi          58%      
node3    42m          2%     1676Mi          62%      
node4    49m          2%     1880Mi          70%
```
### 测试动态扩容功能
- 创建测试demo
``` yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo
spec:
  selector:
    matchLabels:
      app: demo
  replicas: 1
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - name: demo
        image: nginx:latest
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 80
        resources:
          requests:
            cpu: 10m
            memory: 100m
---
apiVersion: v1
kind: Service
metadata:
  name: demo-svc
  labels:
    app: demo
spec:
  selector:
    app: demo
  type: NodePort
  ports:
  - name: http-port
    port: 80
    targetPort: http
    nodePort: 30008
```
- 查看当前pods的副本数和资源使用情况
``` shell
[root@master hpa]#  kubectl top pods
NAME                                     CPU(cores)   MEMORY(bytes)  
demo-d55c586cc-fxxcb                     0m           3Mi
```
- 创建hpa，实现自动扩缩容
``` shell
#kubectl autoscale deployment demo --cpu-percent=10 --min=1 --max=10
指定cpu的指标是10%，超过10%将自动扩容副本。最大副本数是10个
```
- .使用ab工具做测试，使负载升高
``` shell
ab -n 1000 -c 100   http://192.168.120.4:30008/
```
- 监控副本是否自动扩容
``` shell
#kubectl describe deployment demo
```
