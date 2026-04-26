# Примеры Kubernetes-манифестов

Этот документ содержит эталонные шаблоны. Адаптируйте их под свой кластер и структуру `kustomize`.

## 1. Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: messager
```

## 2. ConfigMap и Secret

### Общий ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: messager-config
  namespace: messager
data:
  FRONTEND_BFF_URL: ""
  FRONTEND_BFF_INTERNAL_URL: "http://bff:8080"
  BFF_HTTP_PORT: "8080"
  USER_HTTP_PORT: "8081"
  MSG_HTTP_PORT: "8082"
  USER_SERVICE_URL: "http://user-service:8081"
  MSG_SERVICE_URL: "http://message-service:8082"
  MSG_UPLOADS_DIR: "/app/uploads"
  POSTGRES_DB: "messager"
  POSTGRES_HOST: "postgres"
  POSTGRES_PORT: "5432"
```

### Secret для БД

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: messager-db-secret
  namespace: messager
type: Opaque
stringData:
  POSTGRES_USER: "messager"
  POSTGRES_PASSWORD: "messager"
  USER_DB_DSN: "host=postgres user=messager password=messager dbname=messager_users sslmode=disable"
  MSG_DB_DSN: "host=postgres user=messager password=messager dbname=messager_messages sslmode=disable"
```

## 3. Postgres

### PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: messager
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 5Gi
```

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: messager
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: messager-db-secret
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: messager-db-secret
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: messager-config
                  key: POSTGRES_DB
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
          readinessProbe:
            exec:
              command: ["sh", "-c", "pg_isready -U $POSTGRES_USER"]
            initialDelaySeconds: 10
            periodSeconds: 5
      volumes:
        - name: postgres-data
          persistentVolumeClaim:
            claimName: postgres-pvc
```

### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: messager
spec:
  selector:
    app: postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
```

## 4. Миграции (Jobs)

### migrate-users

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: migrate-users
  namespace: messager
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate-users
          image: ghcr.io/kukymbr/goose-docker:latest
          env:
            - name: GOOSE_DRIVER
              value: postgres
            - name: GOOSE_DBSTRING
              value: "host=postgres user=messager password=messager dbname=messager_users sslmode=disable"
            - name: GOOSE_MIGRATION_DIR
              value: /migrations
          volumeMounts:
            - name: migrations
              mountPath: /migrations
      volumes:
        - name: migrations
          configMap:
            name: users-migrations-cm
```

### migrate-messages

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: migrate-messages
  namespace: messager
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate-messages
          image: ghcr.io/kukymbr/goose-docker:latest
          env:
            - name: GOOSE_DRIVER
              value: postgres
            - name: GOOSE_DBSTRING
              value: "host=postgres user=messager password=messager dbname=messager_messages sslmode=disable"
            - name: GOOSE_MIGRATION_DIR
              value: /migrations
          volumeMounts:
            - name: migrations
              mountPath: /migrations
      volumes:
        - name: migrations
          configMap:
            name: messages-migrations-cm
```

> В реальном решении миграции удобнее запускать как `initContainer` или отдельным CI/CD этапом. Для лабораторной подходит Job.

## 5. user-service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: messager
spec:
  replicas: 1
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
        - name: user-service
          image: mablinov2704/user-service:latest
          ports:
            - containerPort: 8081
          env:
            - name: HTTP_PORT
              valueFrom:
                configMapKeyRef:
                  name: messager-config
                  key: USER_HTTP_PORT
            - name: DB_DSN
              valueFrom:
                secretKeyRef:
                  name: messager-db-secret
                  key: USER_DB_DSN
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: messager
spec:
  selector:
    app: user-service
  ports:
    - name: http
      port: 8081
      targetPort: 8081
```

## 6. message-service (с volume для файлов)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: message-service
  namespace: messager
spec:
  replicas: 1
  selector:
    matchLabels:
      app: message-service
  template:
    metadata:
      labels:
        app: message-service
    spec:
      containers:
        - name: message-service
          image: mablinov2704/message-service:latest
          ports:
            - containerPort: 8082
          env:
            - name: HTTP_PORT
              valueFrom:
                configMapKeyRef:
                  name: messager-config
                  key: MSG_HTTP_PORT
            - name: DB_DSN
              valueFrom:
                secretKeyRef:
                  name: messager-db-secret
                  key: MSG_DB_DSN
            - name: UPLOADS_DIR
              valueFrom:
                configMapKeyRef:
                  name: messager-config
                  key: MSG_UPLOADS_DIR
          volumeMounts:
            - name: uploads
              mountPath: /app/uploads
      volumes:
        - name: uploads
          persistentVolumeClaim:
            claimName: message-uploads-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: message-service
  namespace: messager
spec:
  selector:
    app: message-service
  ports:
    - name: http
      port: 8082
      targetPort: 8082
```

## 7. bff

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bff
  namespace: messager
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bff
  template:
    metadata:
      labels:
        app: bff
    spec:
      containers:
        - name: bff
          image: mablinov2704/bff:latest
          ports:
            - containerPort: 8080
          env:
            - name: HTTP_PORT
              valueFrom:
                configMapKeyRef:
                  name: messager-config
                  key: BFF_HTTP_PORT
            - name: USER_SERVICE_URL
              valueFrom:
                configMapKeyRef:
                  name: messager-config
                  key: USER_SERVICE_URL
            - name: MSG_SERVICE_URL
              valueFrom:
                configMapKeyRef:
                  name: messager-config
                  key: MSG_SERVICE_URL
---
apiVersion: v1
kind: Service
metadata:
  name: bff
  namespace: messager
spec:
  selector:
    app: bff
  ports:
    - name: http
      port: 8080
      targetPort: 8080
```

## 8. frontend + Ingress

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: messager
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: mablinov2704/frontend:latest
          ports:
            - containerPort: 80
          env:
            - name: BFF_URL
              valueFrom:
                configMapKeyRef:
                  name: messager-config
                  key: FRONTEND_BFF_URL
            - name: BFF_INTERNAL_URL
              valueFrom:
                configMapKeyRef:
                  name: messager-config
                  key: FRONTEND_BFF_INTERNAL_URL
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: messager
spec:
  selector:
    app: frontend
  ports:
    - name: http
      port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: messager-ingress
  namespace: messager
spec:
  ingressClassName: nginx
  rules:
    - host: messager.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

## 9. Пример kustomization (base)

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: messager
resources:
  - namespace.yaml
  - configmap.yaml
  - secret.yaml
  - postgres-pvc.yaml
  - postgres.yaml
  - migrate-users-job.yaml
  - migrate-messages-job.yaml
  - user-service.yaml
  - message-service.yaml
  - bff.yaml
  - frontend.yaml
  - ingress.yaml
```
