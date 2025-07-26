# Домашнее задание к занятию «Запуск приложений в K8S»

### Задание 1. Создать Deployment и обеспечить доступ к репликам приложения из другого Pod
1. Создать Deployment приложения, состоящего из двух контейнеров — nginx и multitool. Решить возникшую ошибку.
2. После запуска увеличить количество реплик работающего приложения до 2.
3. Продемонстрировать количество подов до и после масштабирования.
4. Создать Service, который обеспечит доступ до реплик приложений из п.1.
5. Создать отдельный Pod с приложением multitool и убедиться с помощью `curl`, что из пода есть доступ до приложений из п.1.

<details>
  <summary>deployment.yaml</summary>
  
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-multitool-deployment
  labels:
    app: nginx-multitool
spec:
  replicas: 2
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
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
      - name: multitool
        image: praqma/network-multitool
        ports:
        - containerPort: 8080
        env:
        - name: HTTP_PORT
          value: "8080"
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
            cpu: "500m"
```

</details>

<details>
  <summary>service.yaml</summary>
  
```
apiVersion: v1
kind: Service
metadata:
  name: nginx-multitool-service
  labels:
    app: nginx-multitool
spec:
  selector:
    app: nginx-multitool
  ports:
  - name: nginx
    protocol: TCP
    port: 80
    targetPort: 80
  - name: multitool
    protocol: TCP
    port: 8080
    targetPort: 8080
  type: ClusterIP

```

</details>

<details>
  <summary>test-pod.yaml</summary>
  
```
apiVersion: v1
kind: Pod
metadata:
  name: test-multitool-pod
  labels:
    app: test
spec:
  containers:
  - name: multitool
    image: praqma/network-multitool
    command: ["/bin/sh"]
    args: ["-c", "sleep infinity"]
  restartPolicy: Never
```

</details>

![01](https://github.com/Myash-New/Kubernetes/blob/main/03/01.jpg)
![02](https://github.com/Myash-New/Kubernetes/blob/main/03/02.jpg)
![03](https://github.com/Myash-New/Kubernetes/blob/main/03/03.jpg)
![04](https://github.com/Myash-New/Kubernetes/blob/main/03/04.jpg)
![05](https://github.com/Myash-New/Kubernetes/blob/main/03/05.jpg)

------

### Задание 2. Создать Deployment и обеспечить старт основного контейнера при выполнении условий

1. Создать Deployment приложения nginx и обеспечить старт контейнера только после того, как будет запущен сервис этого приложения.
2. Убедиться, что nginx не стартует. В качестве Init-контейнера взять busybox.
3. Создать и запустить Service. Убедиться, что Init запустился.
4. Продемонстрировать состояние пода до и после запуска сервиса.

<details>
  <summary>nginx-deployment.yaml</summary>
  
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      initContainers:
      - name: wait-for-service
        image: busybox:1.35
        command: ['sh', '-c']
        args:
        - |
          echo "Waiting for nginx-service to be available..."
          while ! nslookup nginx-service.default.svc.cluster.local; do
            echo "Service not ready, waiting..."
            sleep 2
          done
          echo "Service is ready, starting nginx..."
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
```

</details>

<details>
  <summary>nginx-service.yaml</summary>
  
```
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: nginx
spec:
  selector:
    app: nginx
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP

 ```

</details>

![06](https://github.com/Myash-New/Kubernetes/blob/main/03/06.jpg)


