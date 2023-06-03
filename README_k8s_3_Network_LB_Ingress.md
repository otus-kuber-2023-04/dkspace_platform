# add probe to pod 

add readinessProbe into kubernetesintro/web-pod.yml from the 1-st lesson

```
...
spec:
  containers:
  - name: web
    image: thatsme/web:1.2
# --- BEGIN ---
    readinessProbe: # Добавим проверку готовности
      httpGet: # веб-сервера отдавать
        path: /index.html # контент
        port: 80
# --- END ---
...
```
check result status is Running

```shell
kubectl apply -f web-pod.yaml
pod/nginx created

kubectl get pod/nginx
NAME    READY   STATUS    RESTARTS   AGE
nginx   0/1     Running   0          2m18s
```
But description ..
```
kubectl describe pod/nginx
Name:             nginx
Namespace:        default
...
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
...

Events:
  Type     Reason     Age                  From               Message
  ----     ------     ----                 ----               -------
..................
  Warning  Unhealthy  96s (x19 over 4m7s)  kubelet            Readiness probe failed: Get "http://10.244.0.30:80/index.html": dial tcp 10.244.0.30:80: connect: connection refused
.................

```



Appply and check result status is Running

```shell
kubectl delete -f web-pod.yaml
pod "nginx" deleted
kubectl apply -f web-pod.yaml
pod/nginx created

kubectl get pod/nginx
NAME    READY   STATUS    RESTARTS   AGE
nginx   0/1     Running   0          2m18s
```
But description ..
```
kubectl describe pod/nginx
Name:             nginx
Namespace:        default
...
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
...

Events:
  Type     Reason     Age                  From               Message
  ----     ------     ----                 ----               -------
..................
  Warning  Unhealthy  96s (x19 over 4m7s)  kubelet            Readiness probe failed: Get "http://10.244.0.30:80/index.html": dial tcp 10.244.0.30:80: connect: connection refused
.................

```

Why below config is meaningless?
Why for some cases it is helpful anyway.

```shell
livenessProbe:
exec:
command:
- 'sh'
- '-c'
- 'ps aux | grep my_web_server_process'

```


To prepare Correct config - delete old pod :

```
kubectl delete pod/web --grace-period=0 --force
# --force also helpful to apply config

k get pods --all-namespaces
NAMESPACE     NAME                               READY   STATUS    RESTARTS       AGE
default       web-748f85d5cc-l85dm               0/1     Running   0              10s
default       web-748f85d5cc-rqkjl               0/1     Running   0              10s
default       web-748f85d5cc-xnp6j               0/1     Running   0              10s
kube-system   coredns-787d4945fb-prwf5           1/1     Running   7 (43m ago)    17d



kubectl delete pods --all -n default --grace-period=0 --force
Warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
pod "web-748f85d5cc-l85dm" force deleted
pod "web-748f85d5cc-rqkjl" force deleted
pod "web-748f85d5cc-xnp6j" force deleted

```

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web   # Название нашего объекта Deployment
spec:
  replicas: 1 # Начнем с одного пода
  selector:    # Укажем, какие поды относятся к нашему Deployment:
    matchLabels:  # - это поды с меткой
      app: web    # app и ее значением web
  template:       # Теперь зададим шаблон конфигурации пода
    metadata:
      labels:
        app: web
    spec:  # Описание Pod
      containers: # Описание контейнеров внутри Pod
      - name: nginx  # Название контейнера
        image: dmikos4/web:testimg  # Образ из которого создается контейнер
        readinessProbe: # Добавим проверку готовности
          httpGet: # веб-сервера отдавать
            path: /index.html # контент
            port: 80
        livenessProbe:
          tcpSocket: { port: 8000 }
        volumeMounts: #У контейнера и у init контейнера должны быть описаны volumeMounts
        - name: app
          mountPath: /app
      initContainers:
      - name: init-web
    #    image: busybox:latest
        image: dmikos4/web:testimg
        command: ['sh', '-c', 'wget -O- https://tinyurl.com/otus-k8s-intro | sh']
        volumeMounts: #У контейнера и у init контейнера должны быть описаны volumeMounts
        - name: app
          mountPath: /app
      volumes: #Для того, чтобы файлы, созданные в init контейнере, были доступны основному контейнеру в pod нам понадобится использовать volume типа emptyDir
      - name: app
        emptyDir: {}


```
Deploy above yaml
```shell
cd kubernetes-networks/
kubectl apply -f web-deploy.yaml --force
deployment.apps/web configured

k get pods --all-namespaces
NAMESPACE     NAME                               READY   STATUS    RESTARTS       AGE
default       web-748f85d5cc-nsmj8               0/1     Running   0              48s
kube-system   coredns-787d4945fb-prwf5           1/1     Running   7 (46m ago)    17d
kube-system   etcd-minikube                      1/1     Running   12 (46m ago)   17d
kube-system   kube-apiserver-minikube            1/1     Running   12 (46m ago)   17d
kube-system   kube-controller-manager-minikube   1/1     Running   12 (46m ago)   17d
kube-system   kube-proxy-z7kbz                   1/1     Running   7 (46m ago)    17d
kube-system   kube-scheduler-minikube            1/1     Running   12 (46m ago)   17d
kube-system   storage-provisioner                1/1     Running   11 (46m ago)   10d

```
Lets check result :
```shell
kubectl describe deployment web
Name:                   web
Namespace:              default
CreationTimestamp:      Tue, 23 May 2023 01:21:50 +0300
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=web
Replicas:               1 desired | 1 updated | 1 total | 0 available | 1 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=web
  Init Containers:
   html-gen:
    Image:      busybox:musl
    Port:       <none>
    Host Port:  <none>
    Command:
      sh
      -c
      wget -O- https://bit.ly/otus-k8s-index-gen | sh
    Environment:  <none>
    Mounts:
      /app from app (rw)
  Containers:
   web:
    Image:        dmikos4/web:testimg
    Port:         <none>
    Host Port:    <none>
    Liveness:     tcp-socket :8000 delay=0s timeout=1s period=10s #success=1 #failure=3
    Readiness:    http-get http://:8000/index.html delay=0s timeout=1s period=10s #success=1 #failure=3
    Environment:  <none>
    Mounts:
      /app from app (rw)
  Volumes:
   app:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
    SizeLimit:  <unset>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      False   MinimumReplicasUnavailable
  Progressing    True    ReplicaSetUpdated
OldReplicaSets:  <none>
NewReplicaSet:   web-748f85d5cc (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  25s   deployment-controller  Scaled down replica set web-748f85d5cc to 1 from 3



```
 Available      False   MinimumReplicasUnavailable

ReadinessProbe not fixed - this is the reason why Deployment status not Ready from not sucessful probe.
As we can see it influence on all Deployment - Available.

To solve it change in web-deploy.yaml:
replicas: 1->3
Port readinessProbe : 8000

```
replicas: 1

readinessProbe: # Добавим проверку готовности
          httpGet: # веб-сервера отдавать
            path: /index.html # контент
            port: 80
```
As result 
```
  k get pods --all-namespaces
NAMESPACE     NAME                               READY   STATUS    RESTARTS       AGE
default       web-5948c498d7-jc2gz               1/1     Running   0              3m26s
default       web-5948c498d7-qw55s               1/1     Running   0              3m57s
default       web-5948c498d7-qx7qs               1/1     Running   0              3m41s
kube-system   coredns-787d4945fb-prwf5           1/1     Running   7 (80m ago)    17d


kubectl describe deployment web
....
  Containers:
   nginx:
    Image:        dmikos4/web:testimg
    Port:         <none>
    Host Port:    <none>
    Liveness:     tcp-socket :8000 delay=0s timeout=1s period=10s #success=1 #failure=3
    Readiness:    http-get http://:8000/index.html delay=0s timeout=1s period=10s #success=1 #failure=3
    Environment:  <none>
    Mounts:
      /app from app (rw)
  Volumes:
   app:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
    SizeLimit:  <unset>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable

```

## Strategy   

test deployment :

maxUnavailable: 0 
maxSurge: 0  

maxUnavailable: 100% 
maxSurge: 100% 

maxUnavailable: 0 
maxSurge: 100% 

maxUnavailable: 100% 
maxSurge: 0 

```
strategy:
 type: RollingUpdate
 rollingUpdate:
  maxUnavailable: 0
  maxSurge: 100%
```
To control deployment use: https://github.com/pulumi/kubespy
```
kubespy trace deploy
or
kubectl get events --watch
```

# Services creation

to allow access to app in k8s from outside

### ClusterIP - most popular type of services

выделяет для каждого сервиса IP-адрес из особого
диапазона (этот адрес виртуален и даже не настраивается на сетевых
интерфейсах)
Когда под внутри кластера пытается подключиться к виртуальному IP-
адресу сервиса, то нода, где запущен под меняет адрес получателя в
сетевых пакетах на настоящий адрес пода.
Нигде в сети, за пределами ноды, виртуальный ClusterIP не
встречается.

ClusterIP use cases:
Нам не надо подключаться к конкретному поду сервиса
Нас устраивается случайное расределение подключений между подами
Нам нужна стабильная точка подключения к сервису, независимая от
подов, нод и DNS-имен
Например:
Подключения клиентов к кластеру БД (multi-read) или хранилищу
Простейшая (не совсем, use IPVS, Luke) балансировка нагрузки внутри
кластера 

For example:
Подключения клиентов к кластеру БД (multi-read) или хранилищу
Простейшая (не совсем, use IPVS, Luke) балансировка нагрузки внутри
кластера

## Service -> ClusterIP

Prepare yaml file /kubernetesnetworks/web-svc-cip.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: web-svc-cip
spec:
  selector:
    app: web
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
```
and apply
```
kubectl apply -f web-svc-cip.yaml
```
result is web-svc-cip   ClusterIP   10.108.37.199
```
kubectl get services
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes    ClusterIP   10.96.0.1       <none>        443/TCP   19d
web-svc-cip   ClusterIP   10.108.37.199   <none>        80/TCP    19s
```
Lets see : curl is OK
```
minikube ssh
Last login: Thu Jun  1 23:52:36 2023 from 192.168.49.1
docker@minikube:~$ sudo -i
root@minikube:~# curl http://10.108.37.199/index.html
<html>
<head/>
...



```

Ping is NOK 
ip addr show - NOK . 
нигде нет ClusterIP

```
 ping 10.108.37.199
PING 10.108.37.199 (10.108.37.199) 56(84) bytes of data.
^C
--- 10.108.37.199 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2118ms

root@minikube:~# arp -an
-bash: arp: command not found

```

```
iptables --list -nv -t nat
```
ClusterIP is in here.

```
Chain KUBE-SVC-6CZTMAROCN3AQODZ (1 references)
 pkts bytes target     prot opt in     out     source               destination
    1    60 KUBE-MARK-MASQ  tcp  --  *      *      !10.244.0.0           10.108.37.199        /* default/web-svc-cip cluster IP */ tcp dpt:80

``` 
Нужное правило находится в цепочке KUBE-SERVICES
Затем мы переходим в цепочку KUBE-SVC-..... - здесь находятся правила
"балансировки" между цепочками KUBE-SEP-.....
SVC - очевидно Service
В цепочках KUBE-SEP-..... находятся конкретные правила
перенаправления трафика (через DNAT)
SEP - Service Endpoint
Подробное описание можно почитать https://msazure.club/kubernetes-services-and-iptables/
или перейти на IPVS, там чуть понятнее ))

# Enable IPVS

from 1.0.0 Minikube support kube-proxy in IPVS mode. Lets try to enable.
При запуске нового инстанса Minikube лучше использовать ключ -
-extra-config и сразу указать, что мы хотим IPVS
Lets enable IPVS for kube-proxy , by changing ConfigMap (конфигурация
Pod, хранящаяся в кластере)
```
kubectl --namespace kube-system edit configmap/kube-proxy
```
Или 
```
minikube dashboard
```
 (далее надо выбрать namespace `kube-system` , Configs and Storage/Config Maps)
Next in kube-proxy config find line `mode:""` and change it like below
```
ipvs:
      strictARP: true
    mode: "ipvs"

```
Удалим Pod с kube-proxy , чтобы применить новую конфигурацию (он входит в DaemonSet и будет запущен автоматически)
Описание работы и настройки . Причины включения strictARP описаны https://github.com/metallb/metallb/issues/153


