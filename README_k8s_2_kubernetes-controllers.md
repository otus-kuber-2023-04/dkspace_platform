# kind cluster started 

```shell
cat kind-config.yaml

kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker


kind create cluster --config kind-config.yaml

$ kubectl get nodes
NAME STATUS ROLES AGE VERSION
kind-control-plane Ready control-plane,master 80s v1.21.1
kind-worker Ready <none> 49s v1.21.1
kind-worker2 Ready <none> 48s v1.21.1
kind-worker3 Ready <none> 50s v1.21.1

```

Make frontend-replicaset.yaml
https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/#example

```shell
cat frontend-replicaset.yaml

apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    app: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: server
        image: dmikos4/frontend:testing
        env:
        - name: PORT
          value: "8080"
        - name: PRODUCT_CATALOG_SERVICE_ADDR
          value: "productcatalogservice:3550"
        - name: CURRENCY_SERVICE_ADDR
          value: "currencyservice:7000"
        - name: CART_SERVICE_ADDR
          value: "cartservice:7070"
        - name: RECOMMENDATION_SERVICE_ADDR
          value: "recommendationservice:8080"
        - name: SHIPPING_SERVICE_ADDR
          value: "shippingservice:50051"
        - name: CHECKOUT_SERVICE_ADDR
          value: "checkoutservice:5050"
        - name: AD_SERVICE_ADDR
          value: "adservice:9555"
        - name: ENABLE_PROFILER
          value: "0"
        resources: {}


kubectl apply -f frontend-replicaset.yaml --force
replicaset.apps/frontend created

kubectl get pods --all-namespaces
NAMESPACE            NAME                                         READY   STATUS    RESTARTS   AGE
default              frontend-x9w58                               1/1     Running   0          11s
#....

kubectl get pods -l app=frontend
NAME             READY   STATUS    RESTARTS   AGE
frontend-x9w58   1/1     Running   0          5m21s

```

Scaling

Одна работающая реплика - это уже неплохо, но в реальной жизни, как
правило, требуется создание нескольких инстансов одного и того же
сервиса для:
Повышения отказоустойчивости;
Распределения нагрузки между репликами.

to set/increase replicas by Ad-hoc cmd
```
kubectl scale replicaset frontend --replicas=3

kubectl get pods -l app=frontend
NAME             READY   STATUS    RESTARTS   AGE
frontend-grgxx   1/1     Running   0          4s
frontend-nc8l6   1/1     Running   0          10m
frontend-z4sph   1/1     Running   0          4s

kubectl get rs frontend
NAME       DESIRED   CURRENT   READY   AGE
frontend   3         3         3       32m
```

Check recovery of pods

Проверим, что благодаря контроллеру pod’ы действительно
восстанавливаются после их ручного удаления:

```shell
kubectl delete pods -l app=frontend | kubectl get pods -l app=frontend -w
NAME             READY   STATUS    RESTARTS   AGE
frontend-6qgtc   1/1     Running   0          30m
frontend-m5555   1/1     Running   0          25s
frontend-nvjqj   1/1     Running   0          25s
frontend-6qgtc   1/1     Terminating   0          30m
frontend-2wjx4   0/1     Pending       0          0s
frontend-m5555   1/1     Terminating   0          25s
frontend-2wjx4   0/1     Pending       0          0s
frontend-psjq8   0/1     Pending       0          0s
frontend-2wjx4   0/1     ContainerCreating   0          0s
frontend-nvjqj   1/1     Terminating         0          25s
frontend-psjq8   0/1     Pending             0          0s
frontend-p98wl   0/1     Pending             0          0s
frontend-p98wl   0/1     Pending             0          0s
frontend-psjq8   0/1     ContainerCreating   0          0s
frontend-p98wl   0/1     ContainerCreating   0          0s
frontend-m5555   0/1     Terminating         0          26s
frontend-m5555   0/1     Terminating         0          26s
frontend-m5555   0/1     Terminating         0          26s
frontend-nvjqj   0/1     Terminating         0          26s
frontend-nvjqj   0/1     Terminating         0          26s
frontend-nvjqj   0/1     Terminating         0          26s
frontend-2wjx4   1/1     Running             0          1s
frontend-6qgtc   0/1     Terminating         0          30m
frontend-6qgtc   0/1     Terminating         0          30m
frontend-6qgtc   0/1     Terminating         0          30m
frontend-psjq8   1/1     Running             0          2s
frontend-p98wl   1/1     Running             0          2s
```


##image check https://kubernetes.io/docs/reference/kubectl/jsonpath/
```shell
 kubectl get replicaset frontend -o=jsonpath='{.spec.template.spec.containers[0].image}'
dmikos4/frontend:testing
```

##image src of managed PODs  
```shell
kubectl get pods -l tier=frontend -o=jsonpath='{.items[0:3].spec.containers[0].image}'
dmikos4/frontend:testing dmikos4/frontend:testing dmikos4/frontend:testingd
```


## Upgrade ReplicaSet

Upgrade/rollout to new version of microservice

```shell
git clone git@github.com:GoogleCloudPlatform/microservices-demo.git

cd microservices-demo/src/frontend/
docker build --rm -t dmikos4/frontend:v0.0.2 .
docker push dmikos4/frontend:v0.0.2

```
Change in frontend-replicaset.yaml 
```
spec:
      containers:
      - name: server
        image: dmikos4/frontend:testing -> dmikos4/frontend:v0.0.2
```
and apply it 

```shell
kubectl apply -f frontend-replicaset.yaml | kubectl get pods -l app=frontend -w

```
and check result of apply - but seems no changes (img still :testing)
```
kubectl get replicaset frontend -o=jsonpath='{.spec.template.spec.containers[0].image}'
dmikos4/frontend:testing

kubectl get pods -l app=frontend -o=jsonpath='{.items[0:3].spec.containers[0].image}'ec.containers[0].image}'
dmikos4/frontend:testing dmikos4/frontend:testing dmikos4/frontend:testing
```

Delete all pods and check - but seems no changes (img still :testing)
```
 kubectl delete pods -l app=frontend | kubectl get pods -l app=frontend -w
NAME             READY   STATUS    RESTARTS   AGE
frontend-49q4w   1/1     Running   0          5m57s
frontend-cggvw   1/1     Running   0          5m57s
frontend-kbzgx   1/1     Running   0          5m57s
frontend-49q4w   1/1     Terminating   0          5m57s
frontend-bhs8l   0/1     Pending       0          0s
#...

kubectl get replicaset frontend -o=jsonpath='{.spec.template.spec.containers[0].image}'
dmikos4/frontend:testing

get pods -l app=frontend -o=jsonpath='{.items[0:3].spec.containers[0].image}'
dmikos4/frontend:testing dmikos4/frontend:testing dmikos4/frontend:testing
```
It means that ReplicaSet not change image because of image deployment.yaml based on :testing version


## Deployment

Prepare images in DockerHub

```shell
cd microservices-demo/src/paymentservice/
docker build --rm -t dmikos4/paymentservice:v0.0.1 .
docker push dmikos4/paymentservice:v0.0.1

docker build --rm -t dmikos4/paymentservice:v0.0.2 .
docker push dmikos4/paymentservice:v0.0.2
```
Prepare paymentservice-replicaset.yaml for dmikos4/paymentservice:v0.0.1 and 3 replics
Prepare deployment.yaml 
```shell
cp paymentservice-replicaset.yaml paymentservice-deployment.yaml
# and kind change ReplicaSet -> Deployment
```
Deployment

```shell
k apply -f paymentservice-deployment.yaml
deployment.apps/paymentservice created

kubectl get deployments
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
paymentservice   3/3     3            3           41s

kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
frontend                    3         3         3       28h
paymentservice-79d9669947   3         3         3       54s

```
## Upgrade 

image version: dmikos4/paymentservice:v0.0.1 -> v0.0.2

```shell

kubectl apply -f paymentservice-deployment.yaml | kubectl get pods -l app=paymentservice -w

NAME                              READY   STATUS    RESTARTS   AGE
paymentservice-79d9669947-49kgw   1/1     Running   0          10m
paymentservice-79d9669947-g64vm   1/1     Running   0          10m
paymentservice-79d9669947-hj26l   1/1     Running   0          10m
paymentservice-6d99776cb-bjdrr    0/1     Pending   0          0s
paymentservice-6d99776cb-bjdrr    0/1     Pending   0          0s
paymentservice-6d99776cb-bjdrr    0/1     ContainerCreating 1new POD   0          0s
paymentservice-6d99776cb-bjdrr    0/1     Running             0          3s
paymentservice-6d99776cb-bjdrr    1/1     Running             0          4s
paymentservice-79d9669947-hj26l   1/1     Terminating  1OLD POD         0          10m
paymentservice-6d99776cb-rhpl8    0/1     Pending             0          0s
paymentservice-6d99776cb-rhpl8    0/1     Pending             0          0s
paymentservice-6d99776cb-rhpl8    0/1     ContainerCreating 2ndNew POD  0          0s
paymentservice-6d99776cb-rhpl8    0/1     Running             0          2s
paymentservice-6d99776cb-rhpl8    1/1     Running             0          3s
paymentservice-79d9669947-g64vm   1/1     Terminating         0          10m
paymentservice-6d99776cb-r4c52    0/1     Pending             0          0s
paymentservice-6d99776cb-r4c52    0/1     Pending             0          0s
paymentservice-6d99776cb-r4c52    0/1     ContainerCreating   0          0s
paymentservice-6d99776cb-r4c52    0/1     Running             0          2s
paymentservice-79d9669947-hj26l   0/1     Terminating         0          10m
paymentservice-79d9669947-hj26l   0/1     Terminating         0          10m
paymentservice-79d9669947-hj26l   0/1     Terminating         0          10m
paymentservice-6d99776cb-r4c52    1/1     Running             0          3s
paymentservice-79d9669947-49kgw   1/1     Terminating         0          10m
paymentservice-79d9669947-g64vm   0/1     Terminating         0          10m
paymentservice-79d9669947-g64vm   0/1     Terminating         0          10m
paymentservice-79d9669947-g64vm   0/1     Terminating         0          10m
paymentservice-79d9669947-49kgw   0/1     Terminating         0          10m
paymentservice-79d9669947-49kgw   0/1     Terminating         0          10m
paymentservice-79d9669947-49kgw   0/1     Terminating         0          10m

kubectl get pods -l app=paymentservice
NAME                             READY   STATUS    RESTARTS   AGE
paymentservice-6d99776cb-bjdrr   1/1     Running   0          4m2s
paymentservice-6d99776cb-r4c52   1/1     Running   0          3m55s
paymentservice-6d99776cb-rhpl8   1/1     Running   0          3m58s

kubectl get pods --all-namespaces
NAMESPACE            NAME                                         READY   STATUS    RESTARTS       AGE
default              frontend-67qfl                               1/1     Running   1 (100m ago)   25h
default              frontend-bhs8l                               1/1     Running   1 (100m ago)   25h
default              frontend-z4jqw                               1/1     Running   1 (100m ago)   25h
default              paymentservice-6d99776cb-bjdrr               1/1     Running   0              6m3s
default              paymentservice-6d99776cb-r4c52               1/1     Running   0              5m56s
default              paymentservice-6d99776cb-rhpl8               1/1     Running   0              5m59s
kube-system          coredns-565d847f94-dg7fp                     1/1     Running   2 (100m ago)   40h
kube-system          coredns-565d847f94-j8gfq                     1/1     Running   2 (100m ago)   40h
kube-system          etcd-kind-control-plane                      1/1     Running   0              100m
kube-system          kindnet-2lssv                                1/1     Running   2 (100m ago)   40h
kube-system          kindnet-9t7wt                                1/1     Running   2 (100m ago)   40h
kube-system          kindnet-mjlrh                                1/1     Running   2 (100m ago)   40h
kube-system          kindnet-ptnq7                                1/1     Running   2 (100m ago)   40h
kube-system          kube-apiserver-kind-control-plane            1/1     Running   0              100m
kube-system          kube-controller-manager-kind-control-plane   1/1     Running   2 (100m ago)   40h
kube-system          kube-proxy-2ncbp                             1/1     Running   2 (100m ago)   40h
kube-system          kube-proxy-j7nkj                             1/1     Running   2 (100m ago)   40h
kube-system          kube-proxy-tmc7r                             1/1     Running   2 (100m ago)   40h
kube-system          kube-proxy-xftzw                             1/1     Running   2 (100m ago)   40h
kube-system          kube-scheduler-kind-control-plane            1/1     Running   2 (100m ago)   40h
local-path-storage   local-path-provisioner-684f458cdd-978hd      1/1     Running   3 (100m ago)   40h

kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
frontend                    3         3         3       28h
paymentservice-6d99776cb    3         3         3       6m29s
paymentservice-79d9669947   0         0         0       17m

# 3 replics
kubectl get rs paymentservice-6d99776cb
NAME                       DESIRED   CURRENT   READY   AGE
paymentservice-6d99776cb   3         3         3       9m47s

# zerro replics
kubernetes-controllers$ kubectl get rs paymentservice-79d9669947
NAME                        DESIRED   CURRENT   READY   AGE
paymentservice-79d9669947   0         0         0       20m

```
## History of upgrade and Rollback

```shell
kubectl rollout history deployment paymentservice
deployment.apps/paymentservice
REVISION  CHANGE-CAUSE
1         <none>
2         <none>

kubectl rollout undo deployment paymentservice --to-revision=1 | kubectl get rs -l
app=paymentservice -w

```

## Deployment maxSure and maxUnavailable

https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy


Аналог blue-green:paymentservice-deployment-bg.yaml
1. Развертывание трех новых pod;
2. Удаление трех старых pod;
```shell

kubectl apply -f paymentservice-deployment-bg.yaml | kubectl get pods -l app=paymentservice -w
NAME                             READY   STATUS    RESTARTS   AGE
paymentservice-6d99776cb-5w6t7   1/1     Running   0          59m
paymentservice-6d99776cb-gwtdq   1/1     Running   0          59m
paymentservice-6d99776cb-jslz5   1/1     Running   0          59m
paymentservice-79d9669947-nt7rg   0/1     Pending   0          0s
paymentservice-79d9669947-l2brb   0/1     Pending   0          0s
paymentservice-79d9669947-cxhsx   0/1     Pending   0          0s
paymentservice-79d9669947-nt7rg   0/1     Pending   0          0s
paymentservice-79d9669947-cxhsx   0/1     Pending   0          0s
paymentservice-79d9669947-l2brb   0/1     Pending   0          0s
paymentservice-79d9669947-nt7rg   0/1     ContainerCreating   0          0s
paymentservice-79d9669947-cxhsx   0/1     ContainerCreating   0          0s
paymentservice-79d9669947-l2brb   0/1     ContainerCreating   0          0s
paymentservice-79d9669947-nt7rg   0/1     Running             0          1s
paymentservice-79d9669947-l2brb   0/1     Running             0          1s
paymentservice-79d9669947-cxhsx   0/1     Running             0          2s

```


Reverse Rolling Update:paymentservice-deployment-reverse.yaml
1. Удаление одного старого pod;
2. Создание одного нового pod;
```shell
kubectl apply -f paymentservice-deployment-reverse.yaml | kubectl get pods -l app=paymentservice -w

kubectl get pods -l app=paymentservice -o=jsonpath='{.items[0:3].spec.containers[0].image}'
dmikos4/paymentservice:v0.0.2 dmikos4/paymentservice:v0.0.2 dmikos4/paymentservice:v0.0.1

```

# Probes
https://kubernetes.io/ru/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

To configure readinessProbe add code into deployment yaml

```
        image: dmikos4/frontend:v0.0.1
        ports:
        - containerPort: 8080
        readinessProbe:
          initialDelaySeconds: 10
          httpGet:
            path: "/_healthz"
            port: 8080
            httpHeaders:
            - name: "Cookie"
              value: "shop_session-id=x-readiness-probe"

```

as result of kubectl apply -f frontend-deployment.yaml
```
k describe pod
Name:             frontend-7fd7795f9d-8wznt
Namespace:        default
Priority:         0
Service Account:  default
Node:             kind-worker/10.10.1.3
Start Time:       Fri, 19 May 2023 20:18:33 +0300
Labels:           app=frontend
...
    Ready:          True
    Restart Count:  1
    Readiness:      http-get http://:8080/_healthz delay=10s timeout=1s period=10s #success=1 #failure=3
...
get pods -l app=frontend
NAME                        READY   STATUS    RESTARTS   AGE
frontend-7fd7795f9d-6dkp4   1/1     Running   0          7m27s
frontend-7fd7795f9d-76wxb   1/1     Running   0          7m27s
frontend-7fd7795f9d-bq6rn   1/1     Running   0          7m27s
```
Imitation of wrong Probe

```
        image: dmikos4/frontend:v0.0.1 -> v0.0.2
        ports:
        - containerPort: 8080
        readinessProbe:
          initialDelaySeconds: 10
          httpGet:
            path: "/_healthz" -> _healt
            port: 8080
            httpHeaders:
            - name: "Cookie"
              value: "shop_session-id=x-readiness-probe"

```
as result of kubectl apply -f frontend-deployment.yaml with wrong probe and new img version =0.0.1:

```
kubectl get pods -l app=frontend
NAME                        READY   STATUS    RESTARTS   AGE
frontend-7fd7795f9d-6dkp4   1/1     Running   0          7m27s
frontend-7fd7795f9d-76wxb   1/1     Running   0          7m27s
frontend-7fd7795f9d-bq6rn   1/1     Running   0          7m27s
frontend-9cffc6875-srgrh    0/1     Running   0          6m25s

# is one not working new POD
k describe pod
...
Name:             frontend-9cffc6875-srgrh
...
    Image:          dmikos4/frontend:v0.0.2
...
    State:          Running
      Started:      Sat, 20 May 2023 13:42:41 +0300
    Ready:          False
...
    Readiness:      http-get http://:8080/_health delay=10s timeout=1s period=10s #success=1 #failure=3
...
Events:
  Type     Reason     Age                  From               Message
  ----     ------     ----                 ----               -------
  Warning  Unhealthy  33s (x102 over 15m)  kubelet            Readiness probe failed: HTTP probe failed with statuscode: 404

```

To check status:

```shell
kubectl rollout status deployment/frontend
error: deployment "frontend" exceeded its progress deadline
```

Example for GitLab CI :
```
deploy_job:
stage: deploy
script:
- kubectl apply -f frontend-deployment.yaml
- kubectl rollout status deployment/frontend --timeout=60s
rollback_deploy_job:
stage: rollback
script:
- kubectl rollout undo deployment/frontend
when: on_failure
```

Norlal status :
```shell

kubectl apply -f frontend-deployment.yaml
deployment.apps/frontend created

k get pod
NAME                        READY   STATUS    RESTARTS   AGE
frontend-7fd7795f9d-756q2   1/1     Running   0          22s
frontend-7fd7795f9d-bmvnl   1/1     Running   0          22s
frontend-7fd7795f9d-rcf7p   1/1     Running   0          22s

kubectl rollout status deployment/frontend
deployment "frontend" successfully rolled out

```

