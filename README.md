# dkspace_platform
dkspace Platform repository

# Building web server inside Docker (Dockerfile) 

1. web-server on port 8000
2. responce content of /app inside Docker. As example if inside /app there is homework.html then after launching it this file should be avilable by URL
http://localhost:8000/homework.html
3 UID 1001

for the reference : https://www.docker.com/blog/how-to-use-the-official-nginx-docker-image/


## how to build Docker

  Create and Make Dockerfile into kubernetes-intro/web
  Build new image

```shell
 docker build --rm -t dmikos4/web:testing .
 docker image
 docker images
 docker save -o web.tar dmikos4/web:testing
#upload to docker hub
 docker push dmikos4/web:testing
```

### Check the result and launch docker

Install kubectl
https://kubernetes.io/ru/docs/reference/kubectl/cheatsheet/#kubectl-autocomplete
https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/
Install minikube
https://minikube.sigs.k8s.io/docs/start/
Install k9s
https://k9scli.io/



```shell
minikube start
kubectl config view
kubectl cluster-info
Kubernetes master is running at https://192.168.99.100:8443
KubeDNS is running at https://192.168.99.100:8443/api/v1/namespaces/kube-system/services/kubedns:
dns/proxy

minikube ssh
docker ps


```
Test Delete all dockers

```shell
docker rm -f $(docker ps -a -q)

```
Test Delete all system PODs
```
kubectl get pods -n kube-system
kubectl delete pod --all -n kube-system
kubectl get componentstatuses
#or
kubectl get cs

NAME STATUS MESSAGE ERROR
controller-manager Healthy ok
scheduler Healthy ok
etcd-0 Healthy {"health":"true"}

kubectl get rs
kubectl get rs --all-namespaces

```

## Prepare kubernetesintro/web-pod.yaml and start/apply

```shell
kubectl apply -f web-pod.yaml

kubectl get pods
NAME READY STATUS RESTARTS AGE
web 1/1 Running 0 42s
```
### Check yaml of running pod

```
kubectl get pod web -o yaml

# 1st step debug running pod

kubectl describe
kubectl describe pod web
```

### Make Init Container

To allow files of Init docker avilable https://kubernetes.io/docs/concepts/workloads/pods/init-containers/
from regular docker add vlume to both of them
```shell
    volumeMounts:
    - name: app
      mountPath: /app
```
volume should be described in pod specifcation
```
  volumes:
  - name: app
    emptyDir: {}
```
To launch 

```shell
kubectl delete pod web
kubectl apply -f web-pod.yaml && kubectl get pods -w
pod/nginx created
NAME    READY   STATUS     RESTARTS   AGE
nginx   0/1     Init:0/1   0          0s
nginx   0/1     Init:0/1   0          7s
nginx   0/1     PodInitializing   0          20s
nginx   1/1     Running           0          21s
```
### Application check


```shell
kubectl port-forward --address 0.0.0.0 pod/nginx 8000:8000
Forwarding from 0.0.0.0:8000 -> 8000
# or use https://kube-forwarder.pixelpoint.io/
```
in browser http://localhost:8000/index.html


## Frontend microservice from Hipster Shop app
https://github.com/GoogleCloudPlatform/microservices-demo

https://github.com/GoogleCloudPlatform/microservices-demo/tree/main/src/frontend

```shell
#https://github.com/GoogleCloudPlatform/microservices-demo.git

git clone git@github.com:GoogleCloudPlatform/microservices-demo.git
cd /microservices-demo/src/frontend
docker build --rm -t dmikos4/frontend:testing .

docker push dmikos4/frontend:testing
The push refers to repository [docker.io/dmikos4/frontend]
```

### Ad-hoc mode to deploy Pod
https://kubernetes.io/docs/reference/kubectl/conventions/
```shell
kubectl run frontend --image dmikos4/frontend:testing --restart=Never
pod/frontend created
#–restart=Never указываем на то, что в качестве ресурса запускаем pod
```
best practice for Ad-hoc mode - to generate yaml
```shell
kubectl run frontend --image dmikos4/frontend:testing --restart=Never --dry-run -o yaml > frontend-pod.yaml
#–dry-run - вывод информации о ресурсе без его реального создания
#-o yaml - форматирование вывода в YAML
#> frontend-pod.yaml - перенаправление вывода в файл
```
Check status

```shell
kubectl get pods -w
NAME       READY   STATUS    RESTARTS   AGE
frontend   0/1     Error     0          46s
nginx      1/1     Running   0          43m
kubectl logs frontend
{"message":"Tracing disabled.","severity":"info","timestamp":"2023-05-15T19:51:48.267700064Z"}
{"message":"Profiling disabled.","severity":"info","timestamp":"2023-05-15T19:51:48.267767586Z"}
panic: environment variable "PRODUCT_CATALOG_SERVICE_ADDR" not set
```
Change frontend-pod.yaml to frontend-pod-healthy.yaml by adding missed env from https://github.com/GoogleCloudPlatform/microservices-demo/blob/main/kubernetes-manifests/frontend.yaml

```shell
kubectl apply -f frontend-pod-healthy.yaml
pod/frontend created
kubectl get pods -w
NAME       READY   STATUS    RESTARTS   AGE
frontend   1/1     Running   0          3s
nginx      1/1     Running   0          59m

```
