# Домашнее задание к занятию «Настройка приложений и управление доступом в Kubernetes»

### Цель задания

Научиться:
- Настраивать конфигурацию приложений с помощью **ConfigMaps** и **Secrets**
- Управлять доступом пользователей через **RBAC**

Это задание поможет вам освоить ключевые механизмы Kubernetes для работы с конфигурацией и безопасностью. Эти навыки необходимы для уверенного администрирования кластеров в реальных проектах. На практике навыки используются для:
- Хранения чувствительных данных (Secrets)
- Гибкого управления настройками приложений (ConfigMaps) 
- Контроля доступа пользователей и сервисов (RBAC)

------


## **Задание 1: Работа с ConfigMaps**
### **Задача**
Развернуть приложение (nginx + multitool), решить проблему конфигурации через ConfigMap и подключить веб-страницу.

### **Шаги выполнения**
1. **Создать Deployment** с двумя контейнерами
   - `nginx`
   - `multitool`
3. **Подключить веб-страницу** через ConfigMap
4. **Проверить доступность**

### **Решение**

<details>
  <summary>configmap-web.yaml</summary>
  
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-webpage
  namespace: default
data:
  index.html: |
    <html>
    <head>
      <title>My Nginx Page</title>
    </head>
    <body>
      <h1>Привет от Kubernetes!</h1>
      <p>Эта страница подключена через ConfigMap.</p>
      <p>Контейнер multitool помогает в диагностике.</p>
    </body>
    </html>
  nginx.conf: |
    events {
    }
    http {
      server {
        listen 80;
        location / {
          root /usr/share/nginx/html;
          index index.html;
        }
      }
    }
```
</details>

<details>
  <summary>deployment.yaml</summary>
  
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-multitool
  labels:
    app: nginx-multitool
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-multitool
  template:
    metadata:
      labels:
        app: nginx-multitool
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: webpage
          mountPath: /usr/share/nginx/html
          readOnly: true
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
          readOnly: true

      - name: multitool
        image: busybox
        command: ["/bin/sh"]
        args: ["-c", "while true; do sleep 3600; done"]
 
      volumes:
      - name: webpage
        configMap:
          name: nginx-webpage
          items:
            - key: index.html
              path: index.html
      - name: nginx-config
        configMap:
          name: nginx-webpage
          items:
            - key: nginx.conf
              path: nginx.conf
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: nginx-multitool
spec:
  type: NodePort
  selector:
    app: nginx-multitool
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
```
</details>

![01](https://github.com/Myash-New/Kubernetes/blob/main/05/01.jpg)
![02](https://github.com/Myash-New/Kubernetes/blob/main/05/02.jpg)
![03](https://github.com/Myash-New/Kubernetes/blob/main/05/03.jpg)

------

## Задание 2. PV, PVC
### Задача
Создать Deployment приложения, использующего локальный PV, созданный вручную.

### Шаги выполнения
1. Создать Deployment приложения, состоящего из контейнеров busybox и multitool, использующего созданный ранее PVC
2. Создать PV и PVC для подключения папки на локальной ноде, которая будет использована в поде.
3. Продемонстрировать, что контейнер multitool может читать данные из файла в смонтированной директории, в который busybox записывает данные каждые 5 секунд. 
4. Удалить Deployment и PVC. Продемонстрировать, что после этого произошло с PV. Пояснить, почему. (Используйте команду `kubectl describe pv`).
5. Продемонстрировать, что файл сохранился на локальном диске ноды. Удалить PV.  Продемонстрировать, что произошло с файлом после удаления PV. Пояснить, почему.

<details>
  <summary>pv-pvc.yaml</summary>
---

apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  volumeName: local-pv
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-exchange-pvc
spec:
  replicas: 1
  selector:
    matchLabels:
      app: data-exchange-pvc
  template:
    metadata:
      labels:
        app: data-exchange-pvc
spec:
  containers:
     - name: busybox-writer
       image: busybox
       command: ["/bin/sh", "-c"]
       args:
         - |
           while true; do
             echo "$(date) - Data written by busybox" >> /shared-data/output.log;
             sleep 5;
           done
       volumeMounts:
          - name: shared-storage
          mountPath: /shared-data
     - name: multitool-reader
       image: gcr.io/google-containers/busybox:1.27
       command: ["/bin/sh", "-c"]
       args:
         - |
           while [ ! -f /shared-data/output.log ]; do
             echo "Waiting for output.log..."
             sleep 2
           done
           tail -f /shared-data/output.log
       volumeMounts:
       - name: shared-storage
         mountPath: /shared-data
     volumes:
     - name: shared-storage
       persistentVolumeClaim:
         claimName: data-pvc
    ```
</details>

![04](https://github.com/Myash-New/Kubernetes/blob/main/05/04.jpg)
Результат:

Status: Released, а не Available.

Reclaim Policy: Retain — значит, PV не удаляется автоматически (политика Retain требует ручного удаления)

Поле Claim становится пустым, но данные на диске остаются:

![05](https://github.com/Myash-New/Kubernetes/blob/main/05/05.jpg)

------

## Задание 3. StorageClass
### Задача
Создать Deployment приложения, использующего PVC, созданный на основе StorageClass.

### Шаги выполнения

1. Создать Deployment приложения, состоящего из контейнеров busybox и multitool, использующего созданный ранее PVC.
2. Создать SC и PVC для подключения папки на локальной ноде, которая будет использована в поде.
3. Продемонстрировать, что контейнер multitool может читать данные из файла в смонтированной директории, в который busybox записывает данные каждые 5 секунд.

<details>
  <summary>sc.yaml</summary>
  
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-sc
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc-sc
spec:
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: local-sc

apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-exchange-sc
spec:
  replicas: 1
  selector:
    matchLabels:
      app: data-exchange-sc
  template:
    metadata:
      labels:
        app: data-exchange-sc
    spec:
      containers:
      - name: busybox-writer
        image: busybox
        command: ["/bin/sh", "-c"]
        args:
          - |
            while true; do
              echo "$(date) - Data written by busybox" >> /shared-data/output.log;
              sleep 5;
            done
        volumeMounts:
        - name: shared-storage
          mountPath: /shared-data

      - name: multitool-reader
        image: gcr.io/google-containers/busybox:1.27
        command: ["/bin/sh", "-c"]
        args:
          - |
            while [ ! -f /shared-data/output.log ]; do
              echo "Waiting for output.log..."
              sleep 2
            done
            tail -f /shared-data/output.log
        volumeMounts:
        - name: shared-storage
          mountPath: /shared-data

      volumes:
      - name: shared-storage
        persistentVolumeClaim:
          claimName: data-pvc-sc
    
    ```
</details>       

<details>
  <summary>pv-manual.yaml</summary>
  
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-sc
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-sc
  hostPath:
    path: /mnt/data-sc
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - vm2
          
    ```
</details> 

![06](https://github.com/Myash-New/Kubernetes/blob/main/05/06.jpg)