# Runbook

## Запуск

```bash
minikube start
kubectl apply -k k8s/overlays/dev
```

Проверка:

```bash
kubectl get pods -n messager
```

Все pod должны быть Running/Completed.

---

## Доступ

Frontend:

http://localhost:30080

---

## Проверка API

Цепочка:

frontend → bff → services

---

## Проверка S3 (файлы)

1. Открыть frontend
2. Отправить файл
3. Убедиться, что файл появился в MinIO (bucket `uploads`)

---

## Node Affinity

* postgres, minio → workload=system
* сервисы → workload=app
* message-service → disk=fast (preferred)

---

## ArgoCD

```bash
kubectl get app -n argocd
```

Ожидается:

```
Synced / Healthy
```


## Подтверждение работы

### Pods

![Pods](images/pods.png)

### ArgoCD

![ArgoCD](images/argocd.png)

### Frontend

![Frontend](images/frontend.png)

### S3 (MinIO)

![S3](images/s3.png)
