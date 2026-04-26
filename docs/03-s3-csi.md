# Подключение S3 через CSI для `message-service`

## Цель

Смонтировать S3-совместимое хранилище в Pod `message-service` как файловую систему по пути `/app/uploads`.

## Варианты хранилища

- Локально: `MinIO` в этом же кластере
- Внешне: AWS S3 / Yandex Object Storage / Selectel S3 / другое S3-совместимое

## Вариант A: MinIO в кластере (рекомендуется для лабораторной)

### 1) Разверните MinIO

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: messager
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
        - name: minio
          image: minio/minio:latest
          args: ["server", "/data", "--console-address", ":9001"]
          ports:
            - containerPort: 9000
            - containerPort: 9001
          env:
            - name: MINIO_ROOT_USER
              valueFrom:
                secretKeyRef:
                  name: minio-secret
                  key: accessKeyID
            - name: MINIO_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: minio-secret
                  key: secretAccessKey
---
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: messager
spec:
  selector:
    app: minio
  ports:
    - name: s3
      port: 9000
      targetPort: 9000
    - name: console
      port: 9001
      targetPort: 9001
```

### 2) Создайте Secret для CSI

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: csi-s3-secret
  namespace: messager
type: Opaque
stringData:
  accessKeyID: "minioadmin"
  secretAccessKey: "minioadmin"
```

### 3) PV/PVC для S3 CSI

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: s3-pv-messager-uploads
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  mountOptions:
    - allow-delete
    - region=us-east-1
    - url=http://minio.messager.svc.cluster.local:9000
    - use_path_request_style
  csi:
    driver: ch.ctrox.csi.s3-driver
    volumeHandle: messager-uploads-handle
    nodePublishSecretRef:
      name: csi-s3-secret
      namespace: messager
    volumeAttributes:
      bucketName: messager-uploads
      mounter: rclone
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: message-uploads-pvc
  namespace: messager
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  volumeName: s3-pv-messager-uploads
```

> Перед тестом создайте bucket `messager-uploads` в MinIO.

### 4) Подключите PVC в `message-service`

`volumeMounts`:

```yaml
volumeMounts:
  - name: uploads
    mountPath: /app/uploads
```

`volumes`:

```yaml
volumes:
  - name: uploads
    persistentVolumeClaim:
      claimName: message-uploads-pvc
```

## Вариант B: Внешний S3

Меняются:

- `url`/`endpoint` в `mountOptions`
- credentials в `Secret`
- регион/параметры адресации

Остальная схема идентична.

## Что проверить

1. Pod `message-service` в статусе `Running`.
2. Внутри контейнера есть каталог `/app/uploads`, смонтированный через CSI.
3. После загрузки картинки через приложение объект появляется в bucket.
4. После перезапуска Pod файлы остаются доступны.

## Частые проблемы

- CSI-driver не установлен в кластере
- неверный `driver` в `PV.spec.csi.driver`
- bucket не создан заранее
- endpoint недоступен из namespace
- неправильные access key / secret key
