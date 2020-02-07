#prometheus 监控node ceph mysql
### prometheus 监控ceph
- 打开ceph prometheus管理模块
ceph mgr module enable prometheus
- 模块开放或，会打开9283端口
- 在k8s master机器上，创建endpoint
``` bash
vi ceph_endpoint.yml
apiVersion: v1
kind: Endpoints
metadata:
  name: ceph-metrics
  namespace: monitoring
  labels:
    k8s-app: ceph-metrics
subsets:
  - addresses:
    - ip: 10.5.19.220
    ports:
    - name: metrics
      port: 9283
      protocol: TCP
```
- 创建servicemonitor资源
``` bash
vi ceph-servicemonitor.yml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  namespace: monitoring
  name: ceph-metrics
  labels:
    k8s-app: ceph-metrics
    app: prometheus-operator-prometheus
    chart: prometheus-operator-8.3.1
    heritage: Helm
    release: prometheus
spec:
  selector:
    matchLabels:
      k8s-app: ceph-metrics
  namespaceSelector:
    matchNames:
    - monitoring
  endpoints:
  - port: metrics
    interval: 10s
    honorLabels: true
```
- 创建service文件
``` bash 
vi  ceph_service.yaml
apiVersion: v1
kind: Service
metadata:
  name: ceph-metrics
  namespace: monitoring
  labels:
    k8s-app: ceph-metrics
spec:
  type: ClusterIP
  ports:
  - name: metrics
    port: 9283
    protocol: TCP
    targetPort: 9283
```
- 部署成功后。打开prometheus dashborad

### 监控node
- 在监控客户机上用docker运行node-exporter
``` bash
docker run -d --net="host" --pid="host" -v "/:/host:ro,rslave" quay.io/prometheus/node-exporter --path.rootfs=/host
```
创建完成后，测试
``` bash
curl 127.0.0.1:9100/metrics
```
- 在prometheus创建监控配置文件
``` bash
[root@pre-k8s-master node]# more node_endpoint.yml
apiVersion: v1
kind: Endpoints
metadata:
  name: node-metrics
  namespace: monitoring
  labels:
    k8s-app: node-metrics
subsets:
  - addresses:
    - ip: 10.5.11.19  #此处是要监控得主机地址
    - ip: 10.5.11.20
    - ip: 10.5.11.21
    ports:
    - name: node
      port: 9100  #此处是node-exporter暴露得服务端口

[root@pre-k8s-master node]# more node_ServiceMonitor.yml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  namespace: monitoring
  name: node-metrics
  labels:
    k8s-app: node-metrics
    app: prometheus-operator-prometheus
    chart: prometheus-operator-8.3.1
    heritage: Helm
    release: prometheus
spec:
  selector:
    matchLabels:
      k8s-app: node-metrics
  namespaceSelector:
    matchNames:
    - monitoring
  endpoints:
  - port: node
    interval: 10s
    honorLabels: true

[root@pre-k8s-master node]# more node_service.yml
apiVersion: v1
kind: Service
metadata:
  name: node-metrics
  namespace: monitoring
  labels:
    k8s-app: node-metrics
spec:
  type: ClusterIP
  ports:
  - name: node
    port: 9100
    protocol: TCP
    targetPort: 9100

#kubectl apply -f .
```
### 监控mysql
- 在mysql主机上用docker启动mysql-exporter
``` bash
docker run -d   --name mysql-6606 --net="host"  -e DATA_SOURCE_NAME="root:tjc8dFX4Nraq3xt8tnrW@(127.0.0.1:6606)/"   prom/mysqld-exporter --web.listen-address=:9104
#--web.listen-address=:9104 此处为设置mysql-exporter的服务端口，默认是9104，如果有多个mysql实例，可以修改不同端口监控
#curl 127.0.0.1:9104/metrics
mysql_up 1 
#此处为1是正常
``` bash
#此处我监控了多个mysql实例
[root@pre-k8s-master mysql]# more mysql_endpoint.yml
apiVersion: v1
kind: Endpoints
metadata:
  name: mysql-metrics
  namespace: monitoring
  labels:
    k8s-app: mysql-metrics
subsets:
  - addresses:
    - ip: 10.5.11.19
    - ip: 10.5.11.20
    - ip: 10.5.11.21
    ports:
    - name: mysql-1
      port: 9104
      protocol: TCP
    - name: mysql-2
      port: 9105
      protocol: TCP
    - name: mysql-3
      port: 9106
      protocol: TCP
    - name: mysql-4
      port: 9107
[root@pre-k8s-master mysql]# more mysql-ServiceMonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  namespace: monitoring
  name: mysql-metrics
  labels:
    k8s-app: mysql-metrics
    app: prometheus-operator-prometheus
    chart: prometheus-operator-8.3.1
    heritage: Helm
    release: prometheus
spec:
  selector:
    matchLabels:
      k8s-app: mysql-metrics
  namespaceSelector:
    matchNames:
    - monitoring
  endpoints:
  - port: mysql-1
    interval: 10s
    honorLabels: true
  - port: mysql-2
    interval: 10s
    honorLabels: true
  - port: mysql-3
    interval: 10s
    honorLabels: true
  - port: mysql-4
    interval: 10s
    honorLabels: true
[root@pre-k8s-master mysql]# more mysql_service.yml
apiVersion: v1
kind: Service
metadata:
  name: mysql-metrics
  namespace: monitoring
  labels:
    k8s-app: mysql-metrics
spec:
  type: ClusterIP
  ports:
  - name: mysql-1
    port: 9104
    protocol: TCP
    targetPort: 9104
  - name: mysql-2
    port: 9105
    protocol: TCP
    targetPort: 9105
  - name: mysql-3
    port: 9106
    protocol: TCP
    targetPort: 9106
  - name: mysql-4
    port: 9107
    protocol: TCP
    targetPort: 9107
```
