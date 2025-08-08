# Домашнее задание к занятию «Компоненты Kubernetes»

### Цель задания

Рассчитать требования к кластеру под проект

------

### Задание. Необходимо определить требуемые ресурсы
Известно, что проекту нужны база данных, система кеширования, а само приложение состоит из бекенда и фронтенда. Опишите, какие ресурсы нужны, если известно:

1. Необходимо упаковать приложение в чарт для деплоя в разные окружения. 
2. База данных должна быть отказоустойчивой. Потребляет 4 ГБ ОЗУ в работе, 1 ядро. 3 копии. 
3. Кеш должен быть отказоустойчивый. Потребляет 4 ГБ ОЗУ в работе, 1 ядро. 3 копии. 
4. Фронтенд обрабатывает внешние запросы быстро, отдавая статику. Потребляет не более 50 МБ ОЗУ на каждый экземпляр, 0.2 ядра. 5 копий. 
5. Бекенд потребляет 600 МБ ОЗУ и по 1 ядру на копию. 10 копий.

### **Решение**

### 🧩 Архитектура приложения

Приложение состоит из 4 компонентов:
1. **База данных**
2. **Система кеширования**
3. **Фронтенд** 
4. **Бэкенд** 

Приложение будет упаковано в Helm-чарт для деплоя в разных окружениях (dev, stage, prod).

##  1. Расчёт ресурсов по компонентам
|-------------------------------------------------------------------------------------------------|
| Компонент       | Кол-во реплик | CPU на реплику | RAM на реплику | Общий CPU   | Общая RAM     |
|-----------------|---------------|----------------|----------------|-------------|---------------|
| **База данных** | 3             | 1 ядро         | 4 ГБ           | 3 ядра      | 12 ГБ         |
| **Кеш**         | 3             | 1 ядро         | 4 ГБ           | 3 ядра      | 12 ГБ         |
| **Фронтенд**    | 5             | 0.2 ядра       | 50 МБ          | 1 ядро      | 250 МБ        |
| **Бэкенд**      | 10            | 1 ядро         | 600 МБ         | 10 ядер     | 6 ГБ          |
| **ИТОГО**       |               |                |                | **17 ядер** | **~30.25 ГБ** |
|-------------------------------------------------------------------------------------------------|

##  2. Отказоустойчивость и запас

Запас на отказоустойчивость (1 нода): Добавляем ресурсы, эквивалентные одной ноде.

Запас на пиковую нагрузку и обновления: Добавляем ~20% к суммарным ресурсам приложения.

### Расчет с учетом запаса:

Приложение + 20% запас:
RAM: 30.25 ГБ * 1.2 = 36.3 ГБ
CPU: 17 ядер * 1.2 = 20.4 ядра

Служебные ресурсы на ноду: (kubelet, containerd, kube-proxy, DaemonSets: logging/monitoring/network)
    RAM: 300 МБ
    CPU: 0.1 ядра

Добавляем ресурсы на системные компоненты (на каждую ноду).

## 3. Выбораем тип и размер нод
 Будем использовать однородные ноды для унификации.
  
Для отказоустойчивости реплики БД и Кеш(StatefulSet) - разнесем на разные ноды, требуется 3 ноды для 3х реплик.
  
  Возьмем размер ноды: 4 vCPU, 16 ГБ RAM

После установки служебных, останется:
    RAM: 16 ГБ - 0.3 ГБ = 15.7 ГБ
    CPU: 4 ядра - 0.1 ядра = 3.9 ядра

## 4. Расчет требуемого количества нод
  
Без запаса:
    RAM: 36.3 / 15.7 ≈ 2.31 -> 3 ноды
    CPU: 20.4 / 3.9 ≈ 5.23 -> **6 нод**

Итог без запаса: Наибольшее значение из расчетов - 6 нод.

Добавление запаса на выход ноды из строя:
  
  Требуется запас на ресурсы одной ноды (15.7 ГБ RAM, 3.9 ядра CPU).
   Новое значение RAM: 36.3 ГБ + 15.7 ГБ = 52 ГБ
   Новое значение CPU: 20.4 ядра + 3.9 ядра = 24.3 ядра

    Расчет:
       RAM: 52 / 15.7 ≈ 3,31 -> 4 ноды
       CPU: 24.3 / 3.9 ≈ 6.23 -> **7 нод** 

# Итог с запасом: 
    
    * Количество рабочих нод: 7

    * Характеристики каждой рабочей ноды:

        vCPU (ядра): 4

        RAM: 16 ГБ




  






3. Планирование нод

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