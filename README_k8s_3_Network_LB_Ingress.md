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
#chat: попробовал так
minikube start --addons=metallb --extra-config kube-proxy.mode=ipvs --extra-config kube-proxy.ipvs.strictARP=true
#ipvs, metallb завелся
```

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
```
kubectl --namespace kube-system delete pod --selector='k8s-app=kube-proxy'
```
Описание работы и настройки . Причины включения strictARP описаны https://github.com/metallb/metallb/issues/153

```
 kubectl get pods --all-namespaces
NAMESPACE              NAME                                        READY   STATUS    RESTARTS      AGE
...
...
kube-system            kube-proxy-6trlp                            1/1     Running   0             12m
...
...

```
Включение IPVS
После успешного рестарта kube-proxy выполним команду
minikube ssh и проверим, что получилось
Выполним команду iptables --list -nv -t nat в ВМ Minikube
Что-то поменялось, но старые цепочки на месте (хотя у них теперь 0
references) 
kube-proxy настроил все по-новому, но не удалил мусор
Запуск kube-proxy --cleanup в нужном поде - тоже не
помогает
kubectl --namespace kube-system exec kube-proxy-<POD> kube-proxy --
cleanup


```
minikube ssh
Last login: Sun Jun  4 01:37:12 2023 from 10.1.1.129
docker@minikube:~$ sudo -i
root@minikube:~# iptables --list -nv -t nat
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
   56  3960 KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */
    3   252 DOCKER_OUTPUT  all  --  *      *       0.0.0.0/0            10.1.1.129          
   27  1620 DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain POSTROUTING (policy ACCEPT 1579 packets, 97031 bytes)
 pkts bytes target     prot opt in     out     source               destination         
 1606 99196 KUBE-POSTROUTING  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes postrouting rules */
    0     0 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0           
    0     0 DOCKER_POSTROUTING  all  --  *      *       0.0.0.0/0            10.1.1.129          
    1    60 CNI-1d638a3757f65620aa54e5ab  all  --  *      *       10.244.0.12          0.0.0.0/0            /* name: "bridge" id: "9c10a3e0af22879e5daf5d6390af7fd6b4ae7790a634565a2fc6d3cc3864a498" */
   12   896 CNI-2a6da5bf36b611dc40b84772  all  --  *      *       10.244.0.13          0.0.0.0/0            /* name: "bridge" id: "b43d98ef794dd3e9c0c6605c889c747adad2501bc55800cdf3576f1db1f6c9b6" */
    1    60 CNI-83fdd3c8874300c87c61e568  all  --  *      *       10.244.0.14          0.0.0.0/0            /* name: "bridge" id: "a5eb9062867ca20aa936ac00ac4fa0d1f9c649995aa8a1e9bdac44b47cc79375" */
    0     0 CNI-879f5a730ae67d7b020a7cda  all  --  *      *       10.244.0.15          0.0.0.0/0            /* name: "bridge" id: "783e99944c92fd24a257e91c2a005ebfc59c5cb06978052c1dc88f4a07f5a85c" */
   12   896 CNI-750f63cb446d8bf62da66764  all  --  *      *       10.244.0.17          0.0.0.0/0            /* name: "bridge" id: "3e9f46cfb2818c5896725bd4734d59144275e28b65f3753484d6a0675324a9c5" */
   12   896 CNI-0fed1596971cac581d9a42be  all  --  *      *       10.244.0.16          0.0.0.0/0            /* name: "bridge" id: "d6e2ac9861db5e69b076cfded43a498026ca6611eb5582847754a2c45a42491f" */

Chain OUTPUT (policy ACCEPT 1520 packets, 92414 bytes)
 pkts bytes target     prot opt in     out     source               destination         
 1386 85372 KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */
   59  4617 DOCKER_OUTPUT  all  --  *      *       0.0.0.0/0            10.1.1.129          
  641 38460 DOCKER     all  --  *      *       0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain DOCKER_OUTPUT (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            10.1.1.129           tcp dpt:53 to:127.0.0.11:39959
   62  4869 DNAT       udp  --  *      *       0.0.0.0/0            10.1.1.129           udp dpt:53 to:127.0.0.11:40198

Chain DOCKER_POSTROUTING (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 SNAT       tcp  --  *      *       127.0.0.11           0.0.0.0/0            tcp spt:39959 to:10.1.1.129:53
    0     0 SNAT       udp  --  *      *       127.0.0.11           0.0.0.0/0            udp spt:40198 to:10.1.1.129:53

Chain DOCKER (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 RETURN     all  --  docker0 *       0.0.0.0/0            0.0.0.0/0           

Chain KUBE-MARK-DROP (0 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            MARK or 0x8000

Chain KUBE-MARK-MASQ (18 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            MARK or 0x4000

Chain KUBE-POSTROUTING (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MASQUERADE  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* Kubernetes endpoints dst ip:port, source ip for solving hairpin purpose */ match-set KUBE-LOOP-BACK dst,dst,src
   22  1320 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            mark match ! 0x4000/0x4000
    0     0 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            MARK xor 0x4000
    0     0 MASQUERADE  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service traffic requiring SNAT */ random-fully

Chain KUBE-KUBELET-CANARY (0 references)
 pkts bytes target     prot opt in     out     source               destination         

Chain CNI-1d638a3757f65620aa54e5ab (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 ACCEPT     all  --  *      *       0.0.0.0/0            10.244.0.0/16        /* name: "bridge" id: "9c10a3e0af22879e5daf5d6390af7fd6b4ae7790a634565a2fc6d3cc3864a498" */
    1    60 MASQUERADE  all  --  *      *       0.0.0.0/0           !224.0.0.0/4          /* name: "bridge" id: "9c10a3e0af22879e5daf5d6390af7fd6b4ae7790a634565a2fc6d3cc3864a498" */

Chain CNI-2a6da5bf36b611dc40b84772 (1 references)
 pkts bytes target     prot opt in     out     source               destination         
   10   776 ACCEPT     all  --  *      *       0.0.0.0/0            10.244.0.0/16        /* name: "bridge" id: "b43d98ef794dd3e9c0c6605c889c747adad2501bc55800cdf3576f1db1f6c9b6" */
    2   120 MASQUERADE  all  --  *      *       0.0.0.0/0           !224.0.0.0/4          /* name: "bridge" id: "b43d98ef794dd3e9c0c6605c889c747adad2501bc55800cdf3576f1db1f6c9b6" */

Chain CNI-83fdd3c8874300c87c61e568 (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 ACCEPT     all  --  *      *       0.0.0.0/0            10.244.0.0/16        /* name: "bridge" id: "a5eb9062867ca20aa936ac00ac4fa0d1f9c649995aa8a1e9bdac44b47cc79375" */
    1    60 MASQUERADE  all  --  *      *       0.0.0.0/0           !224.0.0.0/4          /* name: "bridge" id: "a5eb9062867ca20aa936ac00ac4fa0d1f9c649995aa8a1e9bdac44b47cc79375" */

Chain CNI-879f5a730ae67d7b020a7cda (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 ACCEPT     all  --  *      *       0.0.0.0/0            10.244.0.0/16        /* name: "bridge" id: "783e99944c92fd24a257e91c2a005ebfc59c5cb06978052c1dc88f4a07f5a85c" */
    0     0 MASQUERADE  all  --  *      *       0.0.0.0/0           !224.0.0.0/4          /* name: "bridge" id: "783e99944c92fd24a257e91c2a005ebfc59c5cb06978052c1dc88f4a07f5a85c" */

Chain CNI-0fed1596971cac581d9a42be (1 references)
 pkts bytes target     prot opt in     out     source               destination         
   10   776 ACCEPT     all  --  *      *       0.0.0.0/0            10.244.0.0/16        /* name: "bridge" id: "d6e2ac9861db5e69b076cfded43a498026ca6611eb5582847754a2c45a42491f" */
    2   120 MASQUERADE  all  --  *      *       0.0.0.0/0           !224.0.0.0/4          /* name: "bridge" id: "d6e2ac9861db5e69b076cfded43a498026ca6611eb5582847754a2c45a42491f" */

Chain CNI-750f63cb446d8bf62da66764 (1 references)
 pkts bytes target     prot opt in     out     source               destination         
   10   776 ACCEPT     all  --  *      *       0.0.0.0/0            10.244.0.0/16        /* name: "bridge" id: "3e9f46cfb2818c5896725bd4734d59144275e28b65f3753484d6a0675324a9c5" */
    2   120 MASQUERADE  all  --  *      *       0.0.0.0/0           !224.0.0.0/4          /* name: "bridge" id: "3e9f46cfb2818c5896725bd4734d59144275e28b65f3753484d6a0675324a9c5" */

Chain KUBE-PROXY-CANARY (0 references)
 pkts bytes target     prot opt in     out     source               destination         

Chain KUBE-SERVICES (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    5   300 RETURN     all  --  *      *       127.0.0.0            0.0.0.0/0           
    0     0 KUBE-MARK-MASQ  all  --  *      *      !10.244.0.0           0.0.0.0/0            /* Kubernetes service cluster ip + port for masquerade purpose */ match-set KUBE-CLUSTER-IP dst,dst
    7   420 KUBE-NODE-PORT  all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL
    0     0 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            match-set KUBE-CLUSTER-IP dst,dst

Chain KUBE-NODEPORTS (0 references)
 pkts bytes target     prot opt in     out     source               destination         

Chain KUBE-SVC-Z6GDYMWE5TV2NNJN (0 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.244.0.0           10.103.128.69        /* kubernetes-dashboard/dashboard-metrics-scraper cluster IP */ tcp dpt:8000
    0     0 KUBE-SEP-K7L7QARSOVKMIWWO  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes-dashboard/dashboard-metrics-scraper -> 10.244.0.15:8000 */

Chain KUBE-SVC-NPX46M4PTMTKRN6Y (0 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.244.0.0           10.96.0.1            /* default/kubernetes:https cluster IP */ tcp dpt:443
    0     0 KUBE-SEP-XGHPBZQ6OE7MORHG  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/kubernetes:https -> 10.1.1.130:8443 */

Chain KUBE-SEP-XGHPBZQ6OE7MORHG (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.1.1.130           0.0.0.0/0            /* default/kubernetes:https */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/kubernetes:https */ tcp to:10.1.1.130:8443

Chain KUBE-SEP-K7L7QARSOVKMIWWO (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.244.0.15          0.0.0.0/0            /* kubernetes-dashboard/dashboard-metrics-scraper */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes-dashboard/dashboard-metrics-scraper */ tcp to:10.244.0.15:8000

Chain KUBE-SVC-ERIFXISQEP7F7OF4 (0 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.244.0.0           10.96.0.10           /* kube-system/kube-dns:dns-tcp cluster IP */ tcp dpt:53
    0     0 KUBE-SEP-T62R2ZCQJ7L2XKT3  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:dns-tcp -> 10.244.0.12:53 */

Chain KUBE-SEP-T62R2ZCQJ7L2XKT3 (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.244.0.12          0.0.0.0/0            /* kube-system/kube-dns:dns-tcp */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:dns-tcp */ tcp to:10.244.0.12:53

Chain KUBE-SVC-JD5MR3NA4I4DYORP (0 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.244.0.0           10.96.0.10           /* kube-system/kube-dns:metrics cluster IP */ tcp dpt:9153
    0     0 KUBE-SEP-TGXK4KBOH5HJRS4R  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:metrics -> 10.244.0.12:9153 */

Chain KUBE-SEP-TGXK4KBOH5HJRS4R (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.244.0.12          0.0.0.0/0            /* kube-system/kube-dns:metrics */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:metrics */ tcp to:10.244.0.12:9153

Chain KUBE-SVC-TCOU7JCQXEZGVUNU (0 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  udp  --  *      *      !10.244.0.0           10.96.0.10           /* kube-system/kube-dns:dns cluster IP */ udp dpt:53
    0     0 KUBE-SEP-LSIUSH75GR7U3TUR  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:dns -> 10.244.0.12:53 */

Chain KUBE-SEP-LSIUSH75GR7U3TUR (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.244.0.12          0.0.0.0/0            /* kube-system/kube-dns:dns */
    0     0 DNAT       udp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:dns */ udp to:10.244.0.12:53

Chain KUBE-SVC-CEZPIJSAUFW5MYPQ (0 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.244.0.0           10.96.168.54         /* kubernetes-dashboard/kubernetes-dashboard cluster IP */ tcp dpt:80
    0     0 KUBE-SEP-RH2CVPYJBJBNUPVX  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes-dashboard/kubernetes-dashboard -> 10.244.0.14:9090 */

Chain KUBE-SEP-RH2CVPYJBJBNUPVX (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.244.0.14          0.0.0.0/0            /* kubernetes-dashboard/kubernetes-dashboard */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes-dashboard/kubernetes-dashboard */ tcp to:10.244.0.14:9090

Chain KUBE-SVC-6CZTMAROCN3AQODZ (0 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.244.0.0           10.100.119.150       /* default/web-svc-cip cluster IP */ tcp dpt:80
    0     0 KUBE-SEP-R4XSX25ZM4W6VXLE  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/web-svc-cip -> 10.244.0.13:8000 */ statistic mode random probability 0.33333333349
    0     0 KUBE-SEP-I6DOMSWWKJXSSIVG  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/web-svc-cip -> 10.244.0.16:8000 */ statistic mode random probability 0.50000000000
    0     0 KUBE-SEP-QILNNAXKGXZVFE4L  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/web-svc-cip -> 10.244.0.17:8000 */

Chain KUBE-SEP-R4XSX25ZM4W6VXLE (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.244.0.13          0.0.0.0/0            /* default/web-svc-cip */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/web-svc-cip */ tcp to:10.244.0.13:8000

Chain KUBE-SEP-I6DOMSWWKJXSSIVG (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.244.0.16          0.0.0.0/0            /* default/web-svc-cip */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/web-svc-cip */ tcp to:10.244.0.16:8000

Chain KUBE-SEP-QILNNAXKGXZVFE4L (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.244.0.17          0.0.0.0/0            /* default/web-svc-cip */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/web-svc-cip */ tcp to:10.244.0.17:8000

Chain KUBE-NODE-PORT (1 references)
 pkts bytes target     prot opt in     out     source               destination         

Chain KUBE-LOAD-BALANCER (0 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       0.0.0.0/0            0.0.0.0/0 
```

#### Полностью очистим все правила iptables :

```
sudo -i
root@minikube:~$ cd /tmp/
root@minikube:/tmp$ touch iptables.cleanup
root@minikube:/tmp$ vi iptables.cleanup 
*nat
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
COMMIT
*filter
COMMIT
*mangle
COMMIT
root@minikube:/tmp$ 
root@minikube:/tmp$ iptables-restore iptables.cleanup
```
Теперь надо подождать (примерно 30 секунд), пока kube-proxy восстановит правила для сервисов
Проверим результат : `iptables --list -nv -t nat`

```
docker@minikube:/tmp$ sudo su -
root@minikube:~# iptables --list -nv -t nat
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
   35  2100 KUBE-POSTROUTING  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes postrouting rules */
    0     0 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0           

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
   35  2100 KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */

Chain KUBE-SERVICES (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    4   240 RETURN     all  --  *      *       127.0.0.0            0.0.0.0/0           
    0     0 KUBE-MARK-MASQ  all  --  *      *      !10.244.0.0           0.0.0.0/0            /* Kubernetes service cluster ip + port for masquerade purpose */ match-set KUBE-CLUSTER-IP dst,dst
   14   840 KUBE-NODE-PORT  all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL
    0     0 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            match-set KUBE-CLUSTER-IP dst,dst

Chain KUBE-POSTROUTING (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MASQUERADE  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* Kubernetes endpoints dst ip:port, source ip for solving hairpin purpose */ match-set KUBE-LOOP-BACK dst,dst,src
   34  2040 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            mark match ! 0x4000/0x4000
    0     0 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            MARK xor 0x4000
    0     0 MASQUERADE  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service traffic requiring SNAT */ random-fully

Chain KUBE-NODE-PORT (1 references)
 pkts bytes target     prot opt in     out     source               destination         

Chain KUBE-LOAD-BALANCER (0 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       0.0.0.0/0            0.0.0.0/0           

Chain KUBE-MARK-MASQ (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            MARK or 0x4000
```
Итак, лишние правила удалены и мы видим только актуальную конфигурацию
kube-proxy периодически делает полную синхронизацию правил в своих цепочках)

Как посмотреть конфигурацию IPVS? Ведь в ВМ нет утилиты ipvsadm ?
Нужно в minikube ставить ipvsadm 
 `sudo apt install ipvsadm`

В ВМ выполним команду toolbox - в результате мы окажется в контейнере с Fedora
Теперь установим ipvsadm : `dnf install -y ipvsadm && dnf clean all`


```
root@minikube:~# ip addr show kube-ipvs0
7: kube-ipvs0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default
    link/ether ce:1b:b3:43:fb:20 brd ff:ff:ff:ff:ff:ff
    inet 10.96.0.10/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.103.128.69/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.96.168.54/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.96.0.1/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.100.119.150/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
root@minikube:~#

```
Выполним ipvsadm --list -n и среди прочих сервисов найдем
наш:
```
ipvsadm --list -nIP Virtual Server version 1.2.1 (size=4096) 
Prot LocalAddress:Port Scheduler Flags  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn 
TCP  10.96.0.1:443 rr  -> 10.0.0.2:8443                Masq    1      2          0 
TCP  10.96.0.10:53 rr  -> 10.244.0.2:53                Masq    1      0          0 
TCP  10.96.0.10:9153 rr  -> 10.244.0.2:9153              Masq    1      0          0 
TCP  10.110.105.129:443 rr  -> 10.244.0.7:9443              Masq    1      0          0 
TCP  10.111.205.14:80 rr  -> 10.244.0.4:8000              Masq    1      0          0 
  -> 10.244.0.5:8000              Masq    1      0          0  -> 10.244.0.6:8000              Masq    1      0          0 
UDP  10.96.0.10:53 rr  -> 10.244.0.2:53                Masq    1      0  
```
Теперь выйдем из контейнера toolbox и сделаем ping (нет пинга)
кластерного IP:
$ ipvsadm --list -n
TCP 10.106.18.171:80 rr
-> 172.17.0.11:8000 Masq 1 0 0
-> 172.17.0.12:8000 Masq 1 0 0
-> 172.17.0.13:8000 Masq 1 0 0
$ ping -c1 10.106.18.171
PING 10.106.18.171 (10.106.18.171): 56 data bytes
64 bytes from 10.106.18.171: seq=0 ttl=64 time=0.277 ms
--- 10.106.18.171 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 0.227/0.227/0.277 ms
Итак, все работает. Но почему пингуется виртуальный IP?
Все просто - он уже не такой виртуальный. Этот IP теперь есть на
интерфейсе kube-ipvs0 :
Также, правила в iptables построены по-другому. Вместо
цепочки правил для каждого сервиса, теперь используются хэш-
таблицы (ipset). Можете посмотреть их, установив утилиту ipset в
toolbox .

# LoadBalancer и Ingress

## Installation MetalLB

MetalLB позволяет запустить внутри кластера L4-балансировщик,который будет принимать извне запросы к сервисам и раскидывать их между подами. Установка его проста: https://metallb.universe.tf/installation/

```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-native.yaml
```

В продуктиве так делать не надо. Сначала стоит скачать файл и
разобраться, что там внутри


```
kubectl --namespace metallb-system get all
NAME                              READY   STATUS    RESTARTS   AGE
pod/controller-5fd797fbf7-vxhgw   1/1     Running   0          96s
pod/speaker-t4vkw                 1/1     Running   0          96s

NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/webhook-service   ClusterIP   10.110.21.224   <none>        443/TCP   96s

NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/speaker   1         1         1       1            1           kubernetes.io/os=linux   96s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/controller   1/1     1            1           96s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/controller-5fd797fbf7   1         1         1       96s


```

https://metallb.universe.tf/configuration/
Теперь настроим балансировщик с помощью ConfigMap

metallb-config.yaml
```
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.17.255.1-172.17.255.255 

```
В конфигурации мы настраиваем:
Режим L2 (анонс адресов балансировщиков с помощью ARP)
Создаем пул адресов 172.17.255.1 - 172.17.255.255 - они будут
назначаться сервисам с типом LoadBalancer
```
kubectl apply -f metallb-config.yaml
configmap/config created
```

### MetalLB -Проверка конфигурации
cp web-svc-cip.yaml web-svc-lb.yaml пропишите конфигурацию
```
cat web-svc-lb.yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
```

```
kubectl apply -f web-svc-lb.yaml
service/web-svc-lb created
```

Теперь посмотрите логи пода-контроллера MetalLB (подставьте
правильное имя!)

`kubectl --namespace metallb-system logs controller-5fd797fbf7-vxhgw| grep ipAll`
```
{"caller":"service.go:142","event":"ipAllocated","ip":["172.17.255.1"],"level":"info","msg":"IP address assigned by controller","ts":"2023-06-06T00:49:38Z"}
```
Обратите внимание на назначенный IP-адрес  ipAllocated","ip":["172.17.255.1"]

In case problem with Balanser (EXTERNAL-IP of EXTERNAL-IP from wrong subnet ) - delete controller :
```
 kubectl get pods --namespace metallb-system
NAME                          READY   STATUS    RESTARTS      AGE
controller-5fd797fbf7-bbtbp   1/1     Running   0             24m
speaker-t4vkw                 1/1     Running   2 (34m ago)   11d
```
`kubectl delete pods controller-5fd797fbf7-vxhgw --namespace metallb-system`


```
kubectl get svc --all-namespaces
NAMESPACE        NAME              TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                  AGE
default          kubernetes        ClusterIP      10.96.0.1       <none>           443/TCP                  27h
default          web-svc-cip       ClusterIP      10.111.205.14   <none>           80/TCP                   27h
default          web-svc-lb        LoadBalancer   10.96.237.41    172.17.255.1   80:30222/TCP             25h
kube-system      kube-dns          ClusterIP      10.96.0.10      <none>           53/UDP,53/TCP,9153/TCP   27h
metallb-system   webhook-service   ClusterIP      10.110.21.224   <none>           443/TCP                  19m
```

(или посмотрите его в выводе `kubectl describe svc web-svc-lb` )

```
kubectl describe svc web-svc-lb
Name:                     web-svc-lb
Namespace:                default
Labels:                   <none>
Annotations:              metallb.universe.tf/ip-allocated-from-pool: example
Selector:                 app=web
Type:                     LoadBalancer
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.96.237.41
IPs:                      10.96.237.41
LoadBalancer Ingress:     172.17.255.1
Port:                     <unset>  80/TCP
TargetPort:               8000/TCP
NodePort:                 <unset>  30222/TCP
Endpoints:                10.244.0.12:8000,10.244.0.13:8000,10.244.0.9:8000
Session Affinity:         None
External Traffic Policy:  Cluster
Events:
  Type     Reason            Age                From                Message
  ----     ------            ----               ----                -------
  Warning  AllocationFailed  25h (x2 over 25h)  metallb-controller  Failed to allocate IP for "default/web-svc-lb": no available IPs
  Warning  AllocationFailed  20m (x2 over 20m)  metallb-controller  Failed to allocate IP for "default/web-svc-lb": no available IPs
  Normal   IPAllocated       12m                metallb-controller  Assigned IP ["172.17.255.1"]
  Normal   nodeAssigned      12m                metallb-speaker     announcing from node "minikube" with protocol "layer2"


```
Если мы попробуем открыть URL
http://<our_LB_address>/index.html , то... ничего не выйдет.
Это потому, что сеть кластера изолирована от нашей основной ОС (а ОС
не знает ничего о подсети для балансировщиков)
Чтобы это поправить, добавим статический маршрут
В реальном окружении это решается добавлением нужной подсети
на интерфейс сетевого оборудования
Или использованием L3-режима (что потребует усилий от
сетевиков, но более предпочтительно)

Найдите IP-адрес виртуалки с Minikube. Например так:
```
dmik@nmslab:~$ minikube ssh
Last login: Sun Jun  4 22:00:45 2023 from 10.0.0.1
docker@minikube:~$ ip addr show eth0
9: eth0@if10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:0a:00:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.2/24 brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
```
или
```
 minikube ip
10.0.0.2
```
```
sudo route add -net 172.17.255.0/24 gw 10.0.0.2
curl http://172.17.255.1/index.html                             <html>
<head/>
<body>
<!-- IMAGE BEGINS HERE -->
<font size="-3">
<pre><font color=white>0111010011111011110010000111011000001110000110010011101000001100101011110010100111010001111101001011000001110110101110111001000110</font><br><font color=white>1001000000100011101100010111010111001011010111111001101101100100111111101101101001001111010111100111101010010011011010010100111110</font><br><font color=white>1001100000000010</font><font color=#fffdfc>1</font><font color=#fffcfb>001000</font><font color=#fefcfb>0001</font><font color=#fffcfb>010</font><font color=#fffdfd>0</font><font color=#fffefe>1</font><font color=#fffffe>1</font><font color=white>01111000101001011010111000100110110101011110101101011

ping 172.17.255.1
PING 172.17.255.1 (172.17.255.1) 56(84) bytes of data.
From 10.0.0.2 icmp_seq=1 Destination Host Unreachable
From 10.0.0.2 icmp_seq=2 Destination Host Unreachable
From 10.0.0.2 icmp_seq=3 Destination Host Unreachable
From 10.0.0.2 icmp_seq=4 Destination Host Unreachable
From 10.0.0.2 icmp_seq=5 Destination Host Unreachable
From 10.0.0.2 icmp_seq=6 Destination Host Unreachable
^C
--- 172.17.255.1 ping statistics ---
8 packets transmitted, 0 received, +6 errors, 100% packet loss, time 7457ms
pipe 4

```


MetalLB | Проверка конфигурации
Если пообновлять страничку с помощью Ctrl-F5 (т.е. игнорируя кэш),
то будет видно, что каждый наш запрос приходит на другой под. Причем,
порядок смены подов - всегда один и тот же.
Так работает IPVS - по умолчанию он использует rr (Round-Robin)
балансировку.
К сожалению, выбрать алгоритм на уровне манифеста сервиса нельзя.
К сожалению, выбрать алгоритм на уровне манифеста сервиса нельзя.
Но когда-нибудь, эта полезная фича https://kubernetes.io/blog/2018/07/09/ipvs-based-in-cluster-load-balancing-deep-dive/ появится 
Доступные алгоритмы балансировки описаны здесь и здесь
https://github.com/kubernetes/kubernetes/blob/1cb3b5807ec37490b4582f22d991c043cc468195/pkg/proxy/apis/config/types.go#L185
http://www.linuxvirtualserver.org/docs/scheduling.html


```
dmik@nmslab:~/dkspace_platform/kubernetes-networks$ kubectl get pods --all-namespaces
NAMESPACE        NAME                               READY   STATUS    RESTARTS      AGE
default          web-5948c498d7-h7pv2               1/1     Running   2 (32h ago)   12d
default          web-5948c498d7-mmpr6               1/1     Running   2 (51m ago)   12d
default          web-5948c498d7-pzx4g               1/1     Running   2 (51m ago)   12d
kube-system      coredns-787d4945fb-9kgxc           1/1     Running   2 (51m ago)   12d
kube-system      etcd-minikube                      1/1     Running   2 (32h ago)   12d
kube-system      kube-apiserver-minikube            1/1     Running   2 (51m ago)   12d
kube-system      kube-controller-manager-minikube   1/1     Running   2 (51m ago)   12d
kube-system      kube-proxy-fqfx6                   1/1     Running   2 (32h ago)   12d
kube-system      kube-scheduler-minikube            1/1     Running   2 (32h ago)   12d
kube-system      storage-provisioner                1/1     Running   6 (14m ago)   12d
metallb-system   controller-5fd797fbf7-bbtbp        1/1     Running   0             38m
metallb-system   speaker-t4vkw                      1/1     Running   2 (48m ago)   11d
dmik@nmslab:~/dkspace_platform/kubernetes-networks$
dmik@nmslab:~/dkspace_platform/kubernetes-networks$
dmik@nmslab:~/dkspace_platform/kubernetes-networks$ curl http://172.17.255.1/index.html  | grep web-
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 82999  100 82999    0     0  11.6M      0 --:--:-- --:--:-- --:--:-- 13.1M
export HOSTNAME='web-5948c498d7-h7pv2'
10.244.0.5      web-5948c498d7-h7pv2</pre>
dmik@nmslab:~/dkspace_platform/kubernetes-networks$ curl http://172.17.255.1/index.html  | grep web-
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 82999  100 82999    0     0  34.0M      0 --:--:-- --:--:-- --:--:-- 79.1M
export HOSTNAME='web-5948c498d7-pzx4g'
10.244.0.4      web-5948c498d7-pzx4g</pre>
dmik@nmslab:~/dkspace_platform/kubernetes-networks$ curl http://172.17.255.1/index.html  | grep web-
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 82999  100 82999    0     0  46.2M      0 --:--:-- --:--:-- --:--:-- 79.1M
export HOSTNAME='web-5948c498d7-h7pv2'
10.244.0.5      web-5948c498d7-h7pv2</pre>
dmik@nmslab:~/dkspace_platform/kubernetes-networks$ curl http://172.17.255.1/index.html  | grep web-
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 82999  100 82999    0     0  8037k      0 --:--:-- --:--:-- --:--:-- 9005k
export HOSTNAME='web-5948c498d7-pzx4g'
10.244.0.4      web-5948c498d7-pzx4g</pre>
dmik@nmslab:~/dkspace_platform/kubernetes-networks$

```



### Задание со ⭐️  DNS через MetalLB
Сделайте сервис LoadBalancer , который откроет доступ к CoreDNS
снаружи кластера (позволит получать записи через внешний IP).
Например, nslookup web.default.cluster.local 172.17.255.10 .
Поскольку DNS работает по TCP и UDP протоколам - учтите это в
конфигурации. Оба протокола должны работать по одному и тому же IP-
адресу балансировщика.
Полученные манифесты положите в подкаталог ./coredns
https://metallb.universe.tf/usage/

## Создание Ingress

Теперь, когда у нас есть балансировщик, можно заняться Ingress-
контроллером и прокси:
неудобно, когда на каждый Web-сервис надо выделять свой IP-адрес
а еще хочется балансировку по HTTP-заголовкам (sticky sessions)
Для нашего домашнего задания возьмем почти "коробочный" ingressnginx
от проекта Kubernetes. Это "достаточно хороший" Ingress для
умеренных нагрузок, основанный на OpenResty и пачке Lua-скриптов.

Install
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/deploy.yaml
```

После установки основных компонентов, в инструкции рекомендуется https://kubernetes.github.io/ingress-nginx/deploy/#bare-metal
применить манифест, который создаст NodePort -сервис. Но у нас есть
MetalLB, мы можем сделать круче.

Можно сделать просто `minikube addons enable ingress` , но мы
не ищем легких путей - Создадим файл nginx-lb.yaml c конфигурацией LoadBalancer -
сервиса (работаем в каталоге kubernetes-networks ):

```
kind: Service
apiVersion: v1
metadata:
name: ingress-nginx
namespace: ingress-nginx
labels:
app.kubernetes.io/name: ingress-nginx
app.kubernetes.io/component: controller
spec:
externalTrafficPolicy: Local
type: LoadBalancer
selector:
app.kubernetes.io/name: ingress-nginx
app.kubernetes.io/component: controller
ports:
- { name: http, port: 80, targetPort: http }
- { name: https, port: 443, targetPort: https }
```

Теперь применим созданный манифест и посмотрим на IP-адрес,
назначенный ему MetalLB

```
get svc --all-namespaces
NAMESPACE        NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                      AGE
default          kubernetes                           ClusterIP      10.96.0.1        <none>         443/TCP                      12d
default          web-svc-cip                          ClusterIP      10.111.205.14    <none>         80/TCP                       12d
default          web-svc-lb                           LoadBalancer   10.96.237.41     172.17.255.1   80:30222/TCP                 12d
ingress-nginx    ingress-nginx                        LoadBalancer   10.105.198.238   172.17.255.2   80:32393/TCP,443:30857/TCP   26s
ingress-nginx    ingress-nginx-controller             NodePort       10.98.176.145    <none>         80:31372/TCP,443:31271/TCP   3m47s
ingress-nginx    ingress-nginx-controller-admission   ClusterIP      10.97.35.2       <none>         443/TCP                      3m47s
kube-system      kube-dns                             ClusterIP      10.96.0.10       <none>         53/UDP,53/TCP,9153/TCP       12d
metallb-system   webhook-service                      ClusterIP      10.110.21.224    <none>         443/TCP                      11d
dmik@nmslab:~/dkspace_platform/kubernetes-networks$ kubectl --namespace metallb-system logs pod/controller-5fd797fbf7-bbtbp | grep Alloca
{"caller":"service.go:142","event":"ipAllocated","ip":["172.17.255.2"],"level":"info","msg":"IP address assigned by controller","ts":"2023-06-17T17:51:24Z"}

```

Теперь можно сделать пинг на этот IP-адрес и даже curl
Если видим страничку 404 от OpenResty (или Nginx) - значит работает!


```
ping 172.17.255.2
PING 172.17.255.2 (172.17.255.2) 56(84) bytes of data.
From 10.0.0.2 icmp_seq=1 Destination Host Unreachable
From 10.0.0.2 icmp_seq=2 Destination Host Unreachable
From 10.0.0.2 icmp_seq=3 Destination Host Unreachable
^C
--- 172.17.255.2 ping statistics ---
4 packets transmitted, 0 received, +3 errors, 100% packet loss, time 3074ms
pipe 4
 curl 172.17.255.2
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>

 minikube service list
|----------------|------------------------------------|--------------|-----------------------|
|   NAMESPACE    |                NAME                | TARGET PORT  |          URL          |
|----------------|------------------------------------|--------------|-----------------------|
| default        | kubernetes                         | No node port |                       |
| default        | web-svc-cip                        | No node port |                       |
| default        | web-svc-lb                         |           80 | http://10.0.0.2:30222 |
| ingress-nginx  | ingress-nginx                      | http/80      | http://10.0.0.2:32393 |
|                |                                    | https/443    | http://10.0.0.2:30857 |
| ingress-nginx  | ingress-nginx-controller           | http/80      | http://10.0.0.2:31372 |
|                |                                    | https/443    | http://10.0.0.2:31271 |
| ingress-nginx  | ingress-nginx-controller-admission | No node port |                       |
| kube-system    | kube-dns                           | No node port |                       |
| metallb-system | webhook-service                    | No node port |                       |
|----------------|------------------------------------|--------------|-----------------------|

 kubectl get svc
NAME          TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)        AGE
kubernetes    ClusterIP      10.96.0.1       <none>         443/TCP        12d
web-svc-cip   ClusterIP      10.111.205.14   <none>         80/TCP         12d
web-svc-lb    LoadBalancer   10.96.237.41    172.17.255.1   80:30222/TCP   12d


```

### Connect Web application

Наш Ingress-контроллер не требует ClusterIP для балансировки
трафика
Список узлов для балансировки заполняется из ресурса Endpoints
нужного сервиса (это нужно для "интеллектуальной" балансировки,
привязки сессий и т.п.)
Поэтому мы можем использовать headless-сервис для нашего веб-
приложения.
Скопируйте web-svc-cip.yaml в web-svc-headless.yaml
измените имя сервиса на web-svc
добавьте параметр clusterIP: None

```
cat web-svc-headless.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  selector:
    app: web
  type: ClusterIP
  clusterIP: None
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000


kubectl apply -f web-svc-headless.yaml
service/web-svc created
```
Теперь примените полученный манифест и проверьте, что ClusterIP
для сервиса web-svc действительно не назначен:
` web-svc   ClusterIP      None`

```
kubectl apply -f web-svc-headless.yaml
service/web-svc created

 get svc --all-namespaces
NAMESPACE        NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                      AGE
default          kubernetes                           ClusterIP      10.96.0.1        <none>         443/TCP                      12d
default          web-svc                              ClusterIP      None             <none>         80/TCP                       64s
default          web-svc-cip                          ClusterIP      10.111.205.14    <none>         80/TCP                       12d
default          web-svc-lb                           LoadBalancer   10.96.237.41     172.17.255.1   80:30222/TCP                 12d
ingress-nginx    ingress-nginx                        LoadBalancer   10.105.198.238   172.17.255.2   80:32393/TCP,443:30857/TCP   9m58s
ingress-nginx    ingress-nginx-controller             NodePort       10.98.176.145    <none>         80:31372/TCP,443:31271/TCP   13m
ingress-nginx    ingress-nginx-controller-admission   ClusterIP      10.97.35.2       <none>         443/TCP                      13m
kube-system      kube-dns                             ClusterIP      10.96.0.10       <none>         53/UDP,53/TCP,9153/TCP       12d
metallb-system   webhook-service                      ClusterIP      10.110.21.224    <none>         443/TCP                      11d
```

### Ingress Rules

Теперь настроим наш ingress-прокси, создав манифест с ресурсом
Ingress (файл назовите web-ingress.yaml ):

```
cat web-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /web
        backend:
          serviceName: web-svc
          servicePort: 8000
```

https://kubernetes.io/docs/concepts/services-networking/ingress/
```
cat web-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx-example
  rules:
  - http:
      paths:
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port:
              number: 8000

```

```
kubectl apply -f web-ingress.yaml
ingress.networking.k8s.io/web created

```

Примените манифест и проверьте, что корректно заполнены Address и Backends
```
kubectl describe ingress/web
Name:             web
Labels:           <none>
Namespace:        default
Address:          10.0.0.2
Ingress Class:    nginx
Default backend:  <default>
Rules:
  Host        Path  Backends
  ----        ----  --------
  *
              /web   web-svc:8000 (10.244.0.18:8000,10.244.0.19:8000,10.244.0.20:8000)
Annotations:  nginx.ingress.kubernetes.io/rewrite-target: /
Events:
  Type    Reason  Age                From                      Message
  ----    ------  ----               ----                      -------
  Normal  Sync    80s (x2 over 91s)  nginx-ingress-controller  Scheduled for sync


kubectl describe svc ingress-nginx -n ingress-nginx
Name:                     ingress-nginx
Namespace:                ingress-nginx
Labels:                   app.kubernetes.io/name=ingress-nginx
                          app.kubernetes.io/part-of=ingress-nginx
Annotations:              metallb.universe.tf/ip-allocated-from-pool: first-pool
Selector:                 app.kubernetes.io/component=controller,app.kubernetes.io/instance=ingress-nginx,app.kubernetes.io/name=ingress-nginx
Type:                     LoadBalancer
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.105.198.238
IPs:                      10.105.198.238
LoadBalancer Ingress:     172.17.255.2
Port:                     http  80/TCP
TargetPort:               http/TCP
NodePort:                 http  32393/TCP
Endpoints:                10.244.0.25:80
Port:                     https  443/TCP
TargetPort:               https/TCP
NodePort:                 https  30857/TCP
Endpoints:                10.244.0.25:443
Session Affinity:         None
External Traffic Policy:  Local
HealthCheck NodePort:     31081
Events:
  Type    Reason        Age                   From             Message
  ----    ------        ----                  ----             -------
  Normal  nodeAssigned  10m (x17 over 3h52m)  metallb-speaker  announcing from node "minikube" with protocol "layer2"


kubectl get svc --all-namespaces
NAMESPACE        NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                      AGE
default          kubernetes                           ClusterIP      10.96.0.1        <none>         443/TCP                      12d
default          web-svc                              ClusterIP      None             <none>         80/TCP                       67m
default          web-svc-cip                          ClusterIP      10.111.205.14    <none>         80/TCP                       12d
default          web-svc-lb                           LoadBalancer   10.96.237.41     172.17.255.1   80:30222/TCP                 12d
ingress-nginx    ingress-nginx                        LoadBalancer   10.105.198.238   172.17.255.2   80:32393/TCP,443:30857/TCP   76m
ingress-nginx    ingress-nginx-controller             NodePort       10.98.176.145    <none>         80:31372/TCP,443:31271/TCP   79m
ingress-nginx    ingress-nginx-controller-admission   ClusterIP      10.97.35.2       <none>         443/TCP                      79m
kube-system      kube-dns                             ClusterIP      10.96.0.10       <none>         53/UDP,53/TCP,9153/TCP       12d
metallb-system   webhook-service                      ClusterIP      10.110.21.224    <none>         443/TCP                      11d
```

Теперь можно проверить, что страничка доступна в браузере
( http://<LB_IP>/web/index.html )
Обратите внимание, что обращения к странице тоже балансируются
между Podами. Только сейчас это происходит средствами nginx, а не IPVS

```
curl http://172.17.255.2/web/index.html
<html>
<head/>
<body>
<!-- IMAGE BEGINS HERE -->
<font size="-3">
<pre><font color=white>0111010011111011110010000111011000001110000110010011101000001100101011110010100111010001111101001011000001110110101110111001000110</font><br><font color=white>1001000000100011101100010111010111001011010111111001101101100100111111101101101001001111010111100111101010010011011010010100111110</font><br><font color=white>1001100000000010</font>
```

## Задания со ⭐️ Ingress для Dashboard
Добавьте доступ к kubernetes-dashboard через наш Ingress-прокси:
Cервис должен быть доступен через префикс /dashboard ).
Kubernetes Dashboard должен быть развернут из официального
манифеста. Актуальная ссылка есть тут https://github.com/kubernetes/dashboard
Написанные вами манифесты положите в подкаталог ./dashboard

## Задания со ⭐️  Canary для Ingress
Реализуйте канареечное развертывание с помощью ingress-nginx :
Перенаправление части трафика на выделенную группу подов должно
происходить по HTTP-заголовку.
Документация https://github.com/kubernetes/ingress-nginx/blob/main/docs/user-guide/nginx-configuration/annotations.md#canary
Естественно, что вам понадобятся 1-2 "канареечных" пода.
Написанные манифесты положите в подкаталог ./canary