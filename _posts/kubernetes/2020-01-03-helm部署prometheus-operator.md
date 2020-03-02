# helm部署prometheus-operator  
### 一、promethues基础知识  
该文采用prometheus operator以及helm工具在kubernetes集群上部署。
关于什么是 Prometheus Operator 以及为什么要用 Prometheus Operator？
``` text
Operator 是以软件的方式定义运维过程，是一系列打包、部署和管理 Kubernetes 应用的方法。  
简单  来说就是将运维过程中的手动操作转换为自动化流程，通过 Kubernetes 的 CRD（Custom   Resource Definition）  
将部署前后的相关操作自动化，同时以参数的方式提供了灵活性。而Prometheus Operator   
是 CoreOS 提供的一整套 Prometheus 的 Operator，方便了 Prometheus 的部署。    
```
- Prometheus架构图  
 ![Image text](../img/prometheus.png)
- Prometheus Server  
  Prometheus Server 是监控系统的服务端，服务端通过服务发现的方式，抓取被监控服务的指标，或者通过 pushgateway 的间接抓取，抓取到指标数据后，通过特定的存储引擎进行存储，同时暴露一个 HTTP 服务，提供用 PromQL 来进行数据查询。注意，Prometheus 是定时采样数据，而不是全量数据。 
- Exporter  
  Prometheus 需要服务暴露 http 接口，如果服务本身没有，我们不需要改造服务，可以通过 exporter 来间接获取。Exporter 就充当了 Prometheus 采集的目标，而由各个 exporter 去直接获取指标。目前大多数的服务都有现成的 exporter，我们不需要重复造轮子，拿来用即可，如 MySQL，MongoDB 等，可以参考这里。
- Push Gateway
  Prometheus 采集指标的方式主要有两种，一种是服务端暴露接口（Exporter），由 Prometheus 主动去抓取指标，称为 pull 模式。另一种是服务端主动上报，服务端将指标主动上报至 Push Gateway，Prometheus 再从 Push Gateway 中获取，称为 push 模式。而 Push Gateway 就是 push 模式中重要的中介角色，用于暂存服务端上报的指标，等待 Prometheus 收集。

  为什么要有两种模式呢？我们来比较一下这两种模式的特点。

  Pull 模式：Prometheus 主动抓取的方式，可以由 Prometheus 服务端控制抓取的频率，简单清晰，控制权在 Prometheus 服务端。通过服务发现机制，可以自动接入新服务，去掉下线的服务，无需任何人工干预。对于各种常见的服务，官方或社区有大量 Exporter 来提供指标采集接口，基本无需开发。是官方推荐的方式。

  Push 模式：由服务端主动上报至 Push Gateway，采集最小粒度由服务端决定，等于 Push Gateway 充当了中介的角色，收集各个服务主动上报的指标，然后再由 Prometheus 来采集。但是这样就存在了 Push Gateway 这个性能单点，而且 Push Gateway 也要处理持久化问题，不然宕机也会丢失部分数据。同时需要服务端提供主动上报的功能，可能涉及一些开发改动。不是首选的方式，但是在一些场景下很适用。例如，一些临时性的任务，存在时间可能非常短，如果采用 Pull 模式，可能抓取不到数据。
- Alert Manager
  Alert Manager 是 Prometheus 的报警组件，当 Prometheus 服务端发现报警时，推送 alert 到 Alert Manager，再由 Alert Manager 发送到通知端，如 Email，Slack，微信，钉钉等。Alert Manager 根据相关规则提供了报警的分组、聚合、抑制、沉默等功能。
- Grafana
  Prometheus 提供了一个简单的 web UI 界面，用于查询数据，查看告警、配置等，官方推荐使用另一个开源项目 grafana 来做指标的可视化展示，制作仪表盘等。
### 二、在kubernetes中部署prometheus
- 部署helm
  ``` bash
  #wget https://get.helm.sh/helm-v3.0.0-linux-amd64.tar.gz
  #tar -zxvf helm-v3.0.0-linux-amd64.tar.gz
  #cp linux-amd64/helm /usr/local/bin/
  #添加官方仓库地址
  #helm repo add stable https://kubernetes-charts.storage.googleapis.com
  #查看本地已添加的存储库
  #helm search repo stable
  ``` 
- 创建存储支持
  ``` bash
  # nfs-client.yml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: nfs-client-provisioner

    ---
    kind: ClusterRole
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: nfs-client-provisioner-runner
    rules:
    - apiGroups: [""]
        resources: ["persistentvolumes"]
        verbs: ["get", "list", "watch", "create", "delete"]
    - apiGroups: [""]
        resources: ["persistentvolumeclaims"]
        verbs: ["get", "list", "watch", "update"]
    - apiGroups: ["storage.k8s.io"]
        resources: ["storageclasses"]
        verbs: ["get", "list", "watch"]
    - apiGroups: [""]
        resources: ["events"]
        verbs: ["list", "watch", "create", "update", "patch"]
    - apiGroups: [""]
        resources: ["endpoints"]
        verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]

    ---
    kind: ClusterRoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: run-nfs-client-provisioner
    subjects:
    - kind: ServiceAccount
        name: nfs-client-provisioner
        namespace: default
    roleRef:
      kind: ClusterRole
      name: nfs-client-provisioner-runner
      apiGroup: rbac.authorization.k8s.io
    ---
    kind: Deployment
    apiVersion: apps/v1
    metadata:
      name: nfs-client-provisioner
    spec:
      selector:
        matchLabels:
          app: nfs-client-provisioner
      replicas: 1
      strategy:
        type: Recreate
      template:
        metadata:
          labels:
            app: nfs-client-provisioner
        spec:
          serviceAccountName: nfs-client-provisioner
          containers:
            - name: nfs-client-provisioner
              image: quay.io/external_storage/nfs-client-provisioner:latest
              volumeMounts:
                - name: nfs-client-root
                  mountPath: /persistentvolumes
              env:
                - name: PROVISIONER_NAME
                value: nfs-storage
                - name: NFS_SERVER
                value: 192.168.120.3   #修改为nfsIP地址
                - name: NFS_PATH
                value: /data/monitoring       #路径可自行修改
          volumes:
            - name: nfs-client-root
              nfs:
                server: 192.168.120.3  #修改为nfsIP地址
                path: /data/monitoring       #路径可自行修改
  ```
  ``` bash
  #prometheus-sc.yml
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: prometheus-nfsclass
    provisioner: nfs-storage
    ---
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: grafana-nfsclass
    provisioner: nfs-storage
  ``` 
  #kubectl apply -f .
- 拉取prometheus-operator
  ``` bash
  #helm pull stable/prometheus-operator
  #tar -zxvf prometheus-operator-8.3.3.tgz
  #cd prometheus-operator 
  ``` 
  
- 根据需求修改配置文件
  ``` yaml
  #vi values.yml
  #修改altermanager的服务，改成nodePort类型
    service:
    annotations: {}
    labels: {}
    clusterIP: ""

    ## Port for Alertmanager Service to listen on
    ##
    port: 9093 
    ## Port to expose on each node
    ## Only used if service.type is 'NodePort'
    ##
    nodePort: 30903 修改nodeport端口
    ## List of IP addresses at which the Prometheus server service is available
    ## Ref: https://kubernetes.io/docs/user-guide/services/#external-ips
    ##
    externalIPs: []
    loadBalancerIP: ""
    loadBalancerSourceRanges: []
    ## Service type
    ##
    type: NodePort 
  #修改prometheus的服务，添加nondePort和修改类型为nodePort
    service:
    annotations: {}
    labels: {}
    clusterIP: ""

    ## Port for Prometheus Service to listen on
    ##
    port: 9090

    ## To be used with a proxy extraContainer port
    targetPort: 9090

    ## List of IP addresses at which the Prometheus server service is available
    ## Ref: https://kubernetes.io/docs/user-guide/services/#external-ips
    ##
    externalIPs: []

    ## Port to expose on each node
    ## Only used if service.type is 'NodePort'
    ##
    nodePort: 30090

    ## Loadbalancer IP
    ## Only use if service.type is "loadbalancer"
    loadBalancerIP: ""
    loadBalancerSourceRanges: []
    ## Service type
    ##
    type: NodePort

    sessionAffinity: ""
    #添加上面部署的prometheus的storageclass
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: prometheus-nfsclass 
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi

  ```
  ``` yaml
  #vi charts/grafana/values.yaml
  #修改nodeport类型，并添加grafana的暴露端口
  service:
  type: NodePort
  port: 80
  targetPort: 3000
  nodePort: 30030
    # targetPort: 4181 To be used with a proxy extraContainer
  annotations: {}
  labels: {}
  portName: service
  #配置grafana的存储类
  persistence:
  type: pvc
  enabled: true
  storageClassName: grafana-nfsclass
  accessModes:
    - ReadWriteOnce
  size: 10Gi
  # annotations: {}
  finalizers:
    - kubernetes.io/pvc-protection
  # subPath: ""
  # existingClaim:
  ```

- 部署crds
  ``` yaml
  #kubectl apply -f crds/
  #kubectl create ns monitoring
  ```
- 部署prometheus-operator
  ``` yaml
  #helm install --name=prometheus --namespace=monitoring  ./ --set prometheusOperator.createCustomResource=false
  [root@master1 prometheus-operator]# kubectl get pods -n monitoring
  NAME                                                     READY   STATUS    RESTARTS   AGE
  alertmanager-prometheus-prometheus-oper-alertmanager-0   2/2     Running   0          4h5m
  prometheus-grafana-577fcb45f4-wmsg6                      2/2     Running   0          4h5m
  prometheus-kube-state-metrics-67b765f8b8-dt4b7           1/1     Running   0          4h5m
  prometheus-prometheus-node-exporter-flphx                1/1     Running   0          4h5m
  prometheus-prometheus-node-exporter-m5gjv                1/1     Running   0          4h5m
  prometheus-prometheus-node-exporter-s9xf2                1/1     Running   0          4h5m
  prometheus-prometheus-node-exporter-zsnbm                1/1     Running   0          4h5m
  prometheus-prometheus-oper-operator-6c4d98f8df-8pwcm     2/2     Running   0          4h5m
  prometheus-prometheus-prometheus-oper-prometheus-0       3/3     Running   1          4h5m
  ```
 #查看service暴露端口
  ``` yaml
  [root@master1 prometheus-operator]# kubectl get svc -n monitoring
  NAME                                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
  alertmanager-operated                     ClusterIP   None            <none>        9093/TCP,9094/TCP,9094/UDP   4h7m
  prometheus-grafana                        NodePort    10.96.174.253   <none>        80:30080/TCP                 4h7m
  prometheus-kube-state-metrics             ClusterIP   10.96.246.166   <none>        8080/TCP                     4h7m
  prometheus-operated                       ClusterIP   None            <none>        9090/TCP                     4h7m
  prometheus-prometheus-node-exporter       ClusterIP   10.96.95.92     <none>        9100/TCP                     4h7m
  prometheus-prometheus-oper-alertmanager   NodePort    10.96.44.37     <none>        9093:30903/TCP               4h7m
  prometheus-prometheus-oper-operator       ClusterIP   10.96.188.253   <none>        8080/TCP,443/TCP             4h7m
  prometheus-prometheus-oper-prometheus     NodePort    10.96.96.240    <none>        9090:30090/TCP               4h7m
  ```
现在prometheus+grafana+altermanager就部署成功了

- 填坑
  刚装好后，kube-proxy和etcd无法获取数据
  1. kube-proxy
  ``` yaml
  #kubectl -n kube-system edit cm kube-proxy
  metricsBindAddress: "0.0.0.0:10249"
  ```
  2. etcd
  ``` yaml
  # vi /etc/kubernetes/manifests/etcd.yaml 
  - --listen-metrics-urls=http://0.0.0.0:2381
  ``` 
- 配置grafana。
  ``` bash
  #获取grafana密码  
  #kubectl get secrets prometheus-grafana  -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode;echo 
  ```
  添加监控模板  
  ![Image text](../img/grafana-home.png)  
  ![Image text](../img/grafana-import.png)  
  ![Image text](../img/grafana-show.png)  
- 安装kubernetes插件  
  #kubectl exec -it prometheus-grafana-85b58d9759-q5vjb -c grafana  -n monitoring /bin/bash  
  ``` bash
  bash-5.0$ bin/grafana-cli plugins install grafana-kubernetes-app 
  installing grafana-kubernetes-app @ 1.0.1
  from: https://grafana.com/api/plugins/grafana-kubernetes-app/versions/1.0.1/download
  into: /var/lib/grafana/plugins

    ✔ Installed grafana-kubernetes-app successfully 

    Restart grafana after installing plugins . <service grafana-server restart>

  ```
  #重启pods
  #kubectl delete pods prometheus-grafana-85b58d9759-q5vjb  -n monitoring
  登陆grafana，在红色位置会看到kubernetes插件已经安装成功  
  ![Image text](../img/plugin-kubernetes.png)   
  点进去后，如下图选择链接的图标配置插件  
  ![Image text](../img/grafana-k8s-plugin.png)   
  这里我们可以添加一个新的 Kubernetes 集群，这里需要填写集群的访问地址：https://kubernetes.default，然后比较重要的是集群访问的证书，勾选上TLS Client Auth和With CA Cert这两项。  
  ![Image text](../img/grafana-k8s-plugin-config.png)   
  集群访问的证书文件，用我们访问集群的 kubectl 的配置文件中的证书信息(~/.kube/config)即可，其中属性certificate-authority-data、client-certificate-data、client-key-data就对应这 CA 证书、Client 证书、Client 私钥，不过 config 文件里面的内容是base64编码过后的，所以我们这里填写的时候要做base64解码。
  ``` bash
   # grep certificate-authority-data: /root/.kube/config |awk '{print $2}' |base64 --decode
    -----BEGIN CERTIFICATE-----
    MIICyDCCAbCgAwIBAgIBADANBgkqhkiG9w0BAQsFADAVMRMwEQYDVQQDEwprdWJl
    cm5ldGVzMB4XDTIwMDExMDIyMTI1N1oXDTMwMDEwNzIyMTI1N1owFTETMBEGA1UE
    AxMKa3ViZXJuZXRlczCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBANIH
    JfYG0VKKxgwzFaf84bLDqo2PuO7i1bdA0kb7NNYeT5O26qDgoJUanAxL+k9VmL4t
    JGzcskKcpIFfF+fzzv6ajhj+PLBakejToBiw7+3Nj1WvC0aKWVdC5vwdJa3QZbPE
    g2xXsLV5RPNPTySEIFXwfMvH6WcGJMfwa8LtyiXx7h/GlzlC2m9rNXszO6Hmcq8b
    VjgjLqWS0HiXPx9d6meTUVZlWdcEnUpNK520USfc4mGjvkvomjMQlZJzOlqQxwU9
    Lt0tjTCr6yEmWIFJn2r6ZEor4m620aK2imp78FxS5/YiZNCn6GWGap1BxtblBPhn
    2oUMfO/4H2uSH1wk430CAwEAAaMjMCEwDgYDVR0PAQH/BAQDAgKkMA8GA1UdEwEB
    /wQFMAMBAf8wDQYJKoZIhvcNAQELBQADggEBAGQ1aD+AoeWoQ8DUhDD4YjRyuUmO
    CfgBpmnHNzWHIw22I+RaXM7dFjBPJIzOeCgYmW5SuRoVxpoqbxEjcu5huw+3G+HU
    Wl+qKssLyB4suG1YzNdwQdrSi87BkghD6mjaALVYrNJMJH2JHbxsXOhi7t3zEqdr
    N8O5l6J1aI6B1iB8dFMNCBbnHyYvUyA1Pll65hbIgTAzxiqPwNury8AYG0M1oTZa
    zLu0q3larAcNPyOoXcQ64Uy2DxKsRNdmEw2PSn7e7KHiBfiTssfK73IrZYMHqJel
    ctladMYkZTQLnxJZbwjLikTHnRMYBR0hjgoYYEytDBkHdpY/dUJGxZGa0sM=
    -----END CERTIFICATE-----
   #grep client-certificate-data: /root/.kube/config |awk '{print $2}' |base64 --decode
    -----BEGIN CERTIFICATE-----
    MIIC8jCCAdqgAwIBAgIIQfUUWd6WIZ0wDQYJKoZIhvcNAQELBQAwFTETMBEGA1UE
    AxMKa3ViZXJuZXRlczAeFw0yMDAxMTAyMjEyNTdaFw0yMTAxMDkyMjEyNThaMDQx
    FzAVBgNVBAoTDnN5c3RlbTptYXN0ZXJzMRkwFwYDVQQDExBrdWJlcm5ldGVzLWFk
    bWluMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEApXTQ5LGTjWtabjzb
    EhpYGhXYIziaL/UiO2YEPJEVzfLFaETjuA0HJgc8lowEo9klDu9iap7droEp+CRi
    gcGOxdSOWleCMWqlrZFi284G9zQPhPSfB9rVeKjEI7KHhG4FZwRY0eTtv8tm70AI
    LNadEgR+nHPj/S+dpOzXVtHcDFc4aSiOsr6idGKgWBUVrg/JCr1p3x6wVEeCMkWP
    jBbGKAJtMLZSpfN6cm9kLCMQ5D/LKaK2llQk6kjHZeSfF7CImmSBB20dO/D5KKMG
    VWU7ustHmlpBauxwxgvNpuW5mUkqAdEmhVZBLhjXO3YRO84c02H6yOh3z4vsYKd6
    ZMpPhQIDAQABoycwJTAOBgNVHQ8BAf8EBAMCBaAwEwYDVR0lBAwwCgYIKwYBBQUH
    AwIwDQYJKoZIhvcNAQELBQADggEBABtbdOeZGBVCf8HSCPRDNj0JRZC2E0JJ1oFa
    Atwb6HNC6Po9n+RuINhHqX820gZuYGgtAMMC6IfJL9x3w7hVO9IP8fr5b6WoxIDi
    i3owndSeuGtHIGUioGevOJiPUo1317Tq/pNArkcm1rC9ZhseNTy5Egn8ooPehq8W
    C7Ccj38ASYNODFgPTe06fbc17EZuJGSl+AoCzumiImBFt+LkvyhepiPv2G6vHIHJ
    dNzeiwzf4mdOXWacrIiXosjt9Mz75Xb1oHrnasQUNaKJKipos75i9y7n4idhZpcH
    1YkYhNkROeiUx0Uml6PSNfq0ScsfauU0gKcmTE1cm2r1SVipsSs=
    # grep client-key-data: /root/.kube/config |awk '{print $2}' |base64 --decode
    -----BEGIN RSA PRIVATE KEY-----
    MIIEowIBAAKCAQEApXTQ5LGTjWtabjzbEhpYGhXYIziaL/UiO2YEPJEVzfLFaETj
    uA0HJgc8lowEo9klDu9iap7droEp+CRigcGOxdSOWleCMWqlrZFi284G9zQPhPSf
    B9rVeKjEI7KHhG4FZwRY0eTtv8tm70AILNadEgR+nHPj/S+dpOzXVtHcDFc4aSiO
    sr6idGKgWBUVrg/JCr1p3x6wVEeCMkWPjBbGKAJtMLZSpfN6cm9kLCMQ5D/LKaK2
    llQk6kjHZeSfF7CImmSBB20dO/D5KKMGVWU7ustHmlpBauxwxgvNpuW5mUkqAdEm
    hVZBLhjXO3YRO84c02H6yOh3z4vsYKd6ZMpPhQIDAQABAoIBAEPObdVrWNb6CeNn
    mTgAB98Y2K0Gg1noEhFZq57OrqrqNTCYGuLQo/9Xs+LWmwgy81b9j822MY9Ua8i/
    0IbwtfwpMfWgK0DpPeC11lfSRVSFx15rickES7vZHhirD5KcLCadx3CtaynQx+yA
    x1jo0xIROXoQnHykSOzQCygPdSiw5T5CzFFXI6VqMayKvUZN2G/6SOoqlVwfOyP+
    xJjtIeFdTu/U9rrnm9B2cDcy4VpU9mzbQk6QsTivGPCP+pRnzKfNxVpBfzPugkXp
    aikGoUug1K6xg75T6GNWYISf05aThCtebJvogbD4Yqe9C2r+vzF3xbaey5LK89aE
    pErFGXkCgYEA0OzSfrVH884c/nrWaIelQYGfMXk4peE57SJVIjdykl52Pq6DCA1P
    TZLA+aNYRfkqN5sIKyZueZTWvLv6AGtt5nO5wO5FfkM46HfRcrDWKlDpsgca+K/9
    XWi22oZ1m67FTviIJnYpSVoeQlwDScn8QAxmoNvdUEkMjRmF7K+xX6sCgYEAyryi
    0xTvg7uf1D3G7EcKrChiZqQgMCs3msnh+LRq6N01obwiakLy6ymCwUGjRPMRmqGX
    +qdSzulkYH1WgbszxeX7PkvpeuPsfLLrKKzEf8mFKa6QG6WuhVAJmBv+MWn2E0iM
    4nNwBGotXplCq+q8eMAhAkngaa2+Y6Ea2skenY8CgYEAyGGrzj1+olcYtfcI977K
    AFcbYW8bpvWrAkbAko3NxJb3IE1ufWWOnx9XGNfTDWHaXPTD0+cLZG8dum0sbgSb
    psFt8p5VZrGAvm5VI4R6FKIAUnDALK9W+aU6Y8M8v7HTqldy9ji0hpb5mLn5qWci
    GigKstPNo3g8ooRNT+J+4/sCgYBhUH7P14LItu0XUbgEfm1PCeuB3VJeDMCaGkHP
    wbp3oIc35TiXzR8RZlGPNlSKqipPETYEaQDEtR8reb5mc1d8rNIWqxWdmX5QtQlP
    8u43cKHsyg/+VV8aOrsAOp+fVH977h1CLasdU2IHwIVSRFDHnWG62X9Q+scix5g+
    YL/DXQKBgFadobr40+OvkKZyOjzHxrdX4mvEF/aT65nNAAzWs+s+cg0SaohFTu8N
    ZzGWhk4nfkJCXKQvtaxgbY/qq+/LG7IE2eIunzZGgN0GqfqDN4uJ1/R0BTHHiDg0
    wsGIOwV7vaCCa0ESzDzHKsS9TiuCn5gUyYaAfeZ1BxEs56B9P83d
    -----END RSA PRIVATE KEY-----
    ```
    ![Image text](../img/grafana-k8s-plugin2.png)

