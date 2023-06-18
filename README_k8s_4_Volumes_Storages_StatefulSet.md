# Launch kind kluster

`kind create cluster`

Развернем StatefulSet c [MinIO](https://min.io/) - локальным S3 хранилищем

## Применение StatefulSet

Конфигурация StatefulSet
```
cat minio-statefulset.yaml

apiVersion: apps/v1
kind: StatefulSet
metadata:
  # This name uniquely identifies the StatefulSet
  name: minio
spec:
  serviceName: minio
  replicas: 1
  selector:
    matchLabels:
      app: minio # has to match .spec.template.metadata.labels
  template:
    metadata:
      labels:
        app: minio # has to match .spec.selector.matchLabels
    spec:
      containers:
      - name: minio
        env:
        - name: MINIO_ACCESS_KEY
          value: "minio"
        - name: MINIO_SECRET_KEY
          value: "minio123"
        image: minio/minio:RELEASE.2019-07-10T00-34-56Z
        args:
        - server
        - /data 
        ports:
        - containerPort: 9000
        # These volume mounts are persistent. Each pod in the PetSet
        # gets a volume mounted based on this field.
        volumeMounts:
        - name: data
          mountPath: /data
        # Liveness probe detects situations where MinIO server instance
        # is not working properly and needs restart. Kubernetes automatically
        # restarts the pods if liveness checks fail.
        livenessProbe:
          httpGet:
            path: /minio/health/live
            port: 9000
          initialDelaySeconds: 120
          periodSeconds: 20
  # These are converted to volume claims by the controller
  # and mounted at the paths mentioned above. 
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 10Gi


kubectl apply -f https://raw.githubusercontent.com/express42/otus-platform-snippets/master/Module-02/Kuberenetes-volumes/minio-statefulset.yaml
statefulset.apps/minio created

```

В результате применения конфигурации должно произойти следующее: 

Запуститься под с MinIO
Создаться PVC
Динамически создаться PV на этом PVC с помощью дефолотного
StorageClass

```
kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
minio-0   1/1     Running   0          77s

```

## Применение Headless Service

Для того, чтобы наш StatefulSet был доступен изнутри кластера,
создадим Headless Service
```
cat minio-headlessservice.yaml

apiVersion: v1
kind: Service
metadata:
  name: minio
  labels:
    app: minio
spec:
  clusterIP: None
  ports:
    - port: 9000
      name: minio
  selector:
    app: minio

kubectl apply -f https://raw.githubusercontent.com/express42/otus-platform-snippets/master/Module-02/Kuberenetes-volumes/minio-headless-service.yaml
service/minio created

```

## Проверка работы MinIO

Проверить работу Minio можно с помощью консольного клиента [mc](https://github.com/minio/mc).
Также для проверки ресурсов k8s помогут команды: 

kubectl get statefulsets
kubectl get pods
kubectl get pvc
kubectl get pv
kubectl describe <resource> <resource_name>

```
kubectl get statefulsets
NAME    READY   AGE
minio   1/1     2m56s
kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
minio-0   1/1     Running   0          3m5s
kubectl get pvc
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-minio-0   Bound    pvc-4cba8124-1cec-4358-bcf0-7c6b0cf4e857   10Gi       RWO            standard       3m14s
kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   REASON   AGE
pvc-4cba8124-1cec-4358-bcf0-7c6b0cf4e857   10Gi       RWO            Delete           Bound    default/data-minio-0   standard                3m17s
```

## Задание со *
В конфигурации нашего StatefulSet данные указаны в открытом виде, что
не безопасно.
Поместите данные в [secrets](https://kubernetes.io/docs/concepts/configuration/secret/) и настройте конфигурацию на их
использование.



