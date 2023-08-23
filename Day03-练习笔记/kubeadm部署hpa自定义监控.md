# kubeadm部署hpa自定义监控

[toc]

## yaml文件获取

```
git clone https://github.com/judddd/kubernetes.git
```



## 部署metrics-server

metrics-server是容器集群监控和性能分析工具，HPA、Dashborad、Kubectl top都依赖于metrics-server收集的数据,所以首先得部署metrics-server

```
kubectl apply -f ./metrics.yaml
```

### 检查metrics-server是否部署成功

```
kubectl get pod -n kube-system
```

![](https://pic.downk.cc/item/5f34f2bf14195aa59444418d.jpg)

```
kubectl top node  #观察node监控指标
```

![](https://pic.downk.cc/item/5f34f1be14195aa59443cf1d.jpg)



## 基于核心指标(Core metrics)的自动扩缩容

Core metrics(核心指标)：从 Kubelet、cAdvisor 等获取度量数据，再由metrics-server提供给 Dashboard、HPA 控制器等使用

### 部署podinfo应用

在default命名空间下部署podinfo应用完成HPA测试

```
kubectl apply -f ./podinfo/podinfo-svc.yaml
kubectl apply -f ./podinfo/podinfo-dep.yaml
```

通过service的NodePort端口访问podinfo，http://<K8S_PUBLIC_IP>:31198

![](https://pic.downk.cc/item/5f34fd0514195aa594472d8c.jpg)

### 部署hpa

定义一个hpa yaml，cpu的平均使用率超过80%，内存平均使用超过200Mi时自动扩缩容Pod个数，pod数范围为2到10

```
cat ./podinfo/podinfo-hpa.yaml

apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: podinfo
spec:
  scaleTargetRef:
    apiVersion: extensions/v1beta1
    kind: Deployment
    name: podinfo
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 80
  - type: Resource
    resource:
      name: memory
      targetAverageValue: 200Mi

kubectl apply -f ./podinfo/podinfo-hpa.yaml
```

一段时间后，HPA控制器能够通过metrics server获取CPU和内存的使用

![image-20200813164618198](C:\Users\hh\AppData\Roaming\Typora\typora-user-images\image-20200813164618198.png)

### 使用ab增加负载

为了增加负载，使用ab做负载测试

```
sudo apt-get install apache2-utils

ab -n 1000 -c 100 http://192.168.10.84:31198/  #对http://192.168.10.84:31198/ 进行1000次请求，100个并发请求压力
```

### 观察hpa事件

一段时间后,查看hpa Events事件

```
kubectl describe hpa
```

![](https://pic.downk.cc/item/5f34ffaf14195aa59447e966.jpg)

可以观察到已经将pod动态增加到10个

![](https://pic.downk.cc/item/5f35001814195aa594480da6.jpg)





## 基于自定义指标(Custom metrics)的自动扩缩容

Core metrics(核心指标)只包含node和pod的cpu、内存等，一般来说，核心指标作HPA已经足够，但如果想根据自定义指标:如请求qps/5xx错误数来实现HPA，就需要使用自定义指标了。

为了基于自定义指标进行扩展，需要安装两个组件。一个组件从应用程序中收集metrics，并将他们存储在promethues的时序数据库中。另一个组件扩展k8s自定义metics API，即k8s-prometheus-adapter

![](https://pic.downk.cc/item/5f3501d614195aa59448b438.jpg)

### 部署prometheus

创建命名空间

```
kubectl apply -f namespaces.yaml
```

部署prometheus应用

```
kubectl apply -f ./prometheus
```

观察是否成功部署

```
kubectl get pod -n monitoring
```

![](https://pic.downk.cc/item/5f3503a614195aa59449654e.jpg)

### 部署k8s-prometheus-adapter

生成prometheus-adapter所需的TLS证书

```
sudo apt-get install make #安装gcc的编译器
make certs #通过Makefile生成证书
```

部署Prometheus custom metrics API adapter

```
kubectl apply -f ./custom-metrics-api
```

列出prometheus提供的自定义指标

```
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1" | jq .
```

### 部署podinfo应用

```
kubectl apply -f ./podinfo/podinfo-svc.yaml
kubectl apply -f ./podinfo/podinfo-dep.yaml
```

### 从自定义metrics API中获取每秒请求总数

podinfo应用暴露了一个名为http_requests_total的自定义metric。Prometheas适配器删除_total后缀，并将度量标记为计数器度量(counter metric)

从自定义metrics API中获取每秒请求总数:

```
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/http_requests" | jq .
```

![](https://pic.downk.cc/item/5f36009514195aa5948709b3.jpg)

m代表milli-units，所以901m代表901 milli-requests

### 部署hpa

创建一个HPA，如果请求数量超过每秒10个，将扩容podinfo应用

```
cat ./podinfo/podinfo-hpa-custom.yaml

apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: podinfo
spec:
  scaleTargetRef:
    apiVersion: extensions/v1beta1
    kind: Deployment
    name: podinfo
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metricName: http_requests
      targetAverageValue: 10

kubectl apply -f ./podinfo/podinfo-hpa-custom.yaml
```

一段时间后，HPA从metrics API获取http_requests值

![](https://pic.downk.cc/item/5f36019414195aa594874db2.jpg)

### 使用hey增加负载

```
sudo apt install hey

#以每秒25次的频率请求podinfo应用
hey -n 10000 -q 5 -c 5 http://IP:31198/
```

### 观察hpa事件

一段时间后,查看hpa Events事件

```
kubectl describe hpa
```

![](https://pic.downk.cc/item/5f3607f614195aa594893697.jpg)

可以观察到已经将pod动态增加到6个

![](https://pic.downk.cc/item/5f36083114195aa594895acc.jpg)

