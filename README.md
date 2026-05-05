# Kubernetes GitOps Deployment (Messenger App)

## Описание

В рамках лабораторной работы развернуто микросервисное приложение мессенджера в Kubernetes с использованием:

* Kubernetes (minikube)
* S3-хранилища через CSI (MinIO)
* GitOps-подхода через Argo CD
* Kustomize (base + dev/prod overlays)

---

## Архитектура

В кластере развернуты сервисы:

* frontend
* bff (API gateway)
* user-service
* message-service
* postgres
* minio (S3 storage)
* jobs миграций (users/messages)

---

## Структура репозитория

```
k8s/
  base/
  overlays/
    dev/
    prod/
argocd/
docs/
```

---

## Запуск (локально, minikube)

```bash
minikube start

kubectl apply -k k8s/overlays/dev
```

Проверка:

```bash
kubectl get pods -n messager
```

---

## Доступ к приложению

Frontend доступен через NodePort:

```bash
kubectl get svc -n messager
```

Открыть в браузере:

```
http://localhost:30080
```

---

## S3 (MinIO + CSI)

* Хранилище реализовано через CSI драйвер
* PVC: `uploads-pvc`
* Монтируется в `/uploads` в message-service

! В minikube используется ClusterIP MinIO:

```
http://<minio-cluster-ip>:9000
```

(в проде должен использоваться DNS)

---

## GitOps (Argo CD)

Установка:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f argocd/install.yaml
```

Application:

```bash
kubectl apply -f argocd/application.yaml
```

Проверка:

```bash
kubectl get app -n argocd
```

Статус должен быть:

```
Synced / Healthy
```

---

## Особенности

* Использован nodeAffinity:

  * `workload=system` → postgres, minio
  * `workload=app` → сервисы
  * `message-service` → предпочтение `disk=fast`
* В minikube используется одна нода → label переключается вручную

