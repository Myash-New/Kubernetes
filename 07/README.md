# Домашнее задание к занятию «Helm»

### Цель задания

В тестовой среде Kubernetes необходимо установить и обновить приложения с помощью Helm.

------

### Задание 1. Подготовить Helm-чарт для приложения

1. Необходимо упаковать приложение в чарт для деплоя в разные окружения. 
2. Каждый компонент приложения деплоится отдельным deployment’ом или statefulset’ом.
3. В переменных чарта измените образ приложения для изменения версии.

### **Решение**

<details>
  <summary>Chart.yaml</summary>
  
```
apiVersion: v2
name: myapp
version: 1.0.0
appVersion: "1.0"
description: Helm chart for deploying frontend, backend, and database
type: application
```
</details>

<details>
  <summary>values.yaml</summary>
  
```
images:
  frontend:
    repository: nginx
    tag: 1.25
  backend:
    repository: mycompany/api-server
    tag: 1.4.0
  database:
    repository: postgres
    tag: 15

frontend:
  replicas: 2
  port: 80

backend:
  replicas: 2
  port: 8080

database:
  replicas: 1
  port: 5432
  dataDir: /var/lib/postgresql/data
  storage:
    size: 10Gi
```
</details>

<details>
  <summary>values-dev.yaml</summary>
  
```
images:
  frontend:
    tag: latest
  backend:
    tag: dev-latest
  database:
    tag: 15-alpine

frontend:
  replicas: 1
backend:
  replicas: 1
```
</details>

<details>
  <summary>templates/frontend-deployment.yaml</summary>
  
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-frontend
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.frontend.replicas }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}-frontend
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}-frontend
    spec:
      containers:
      - name: frontend
        image: "{{ .Values.images.frontend.repository }}:{{ .Values.images.frontend.tag }}"
        ports:
        - containerPort: {{ .Values.frontend.port }}
        resources: {}
```
</details>

<details>
  <summary>templates/backend-deployment.yaml</summary>
  
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-backend
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.backend.replicas }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}-backend
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}-backend
    spec:
      containers:
      - name: backend
        image: "{{ .Values.images.backend.repository }}:{{ .Values.images.backend.tag }}"
        ports:
        - containerPort: {{ .Values.backend.port }}
        env:
        - name: DATABASE_URL
          value: "postgresql://{{ .Release.Name }}-database:5432/myapp"
        resources: {}
```
</details>

<details>
  <summary>templates/database-statefulset.yaml</summary>
  
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ .Release.Name }}-database
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  serviceName: {{ .Release.Name }}-database
  replicas: {{ .Values.database.replicas }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}-database
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}-database
    spec:
      containers:
      - name: postgres
        image: "{{ .Values.images.database.repository }}:{{ .Values.database.tag }}"
        ports:
        - containerPort: {{ .Values.database.port }}
        env:
        - name: POSTGRES_DB
          value: myapp
        - name: POSTGRES_USER
          value: admin
        - name: POSTGRES_PASSWORD
          value: password
        volumeMounts:
        - name: data-volume
          mountPath: {{ .Values.database.dataDir }}
  volumeClaimTemplates:
  - metadata:
      name: data-volume
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: {{ .Values.database.storage.size }}
```
</details>


<details>
  <summary>_helpers.tpl</summary>
  
```
{{/*
Common labels
*/}}
{{- define "myapp.labels" -}}
app: {{ .Chart.Name }}
chart: {{ .Chart.Name }}-{{ .Chart.Version }}
release: {{ .Release.Name }}
heritage: {{ .Release.Service }}
{{- end }}
```
</details>

![01](https://github.com/Myash-New/Kubernetes/blob/main/07/01.jpg)

------
### Задание 2. Запустить две версии в разных неймспейсах

1. Подготовив чарт, необходимо его проверить. Запуститe несколько копий приложения.
2. Одну версию в namespace=app1, вторую версию в том же неймспейсе, третью версию в namespace=app2.
3. Продемонстрируйте результат.

### **Решение**

![02](https://github.com/Myash-New/Kubernetes/blob/main/07/02.jpg)
![03](https://github.com/Myash-New/Kubernetes/blob/main/07/03.jpg)