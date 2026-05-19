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

Если профиль minikube уже был создан с одной нодой, перед запуском нужно пересоздать кластер или добавить недостающие ноды.

```bash
minikube start --nodes 3

kubectl label node minikube workload=system --overwrite
kubectl label node minikube-m02 workload=app --overwrite
kubectl label node minikube-m03 workload=app disk=fast --overwrite

kubectl apply -k k8s/overlays/dev
```

Проверка:

```bash
kubectl get nodes --show-labels
kubectl get pods -n messager
```

---

## Доступ к приложению

Frontend доступен через NodePort:

```bash
kubectl get svc -n messager
minikube service frontend -n messager --url
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

Внутри кластера MinIO доступен через DNS сервиса:

```
http://minio.messager.svc.cluster.local:9000
```

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
  * `workload=app` → frontend, bff, user-service, message-service
  * `message-service` → предпочтение `disk=fast`
* Для локального запуска minikube поднимается с 3 нодами:
  * `minikube` → `workload=system`
  * `minikube-m02` → `workload=app`
  * `minikube-m03` → `workload=app`, `disk=fast`

## Дополнительная информация

Подробный отчёт с:

* инструкцией по запуску
* проверками работоспособности
* nodeAffinity
* ArgoCD
* S3 CSI
* скриншотами работы

находится в:

```text
docs/report.md
```
