# Домашнее задание к занятию «Сетевое взаимодействие в K8S. Часть 1»

### Цель задания

В тестовой среде Kubernetes необходимо обеспечить доступ к приложению, установленному в предыдущем ДЗ и состоящему из двух контейнеров, по разным портам в разные контейнеры как внутри кластера, так и снаружи.

------

### Задание 1. Создать Deployment и обеспечить доступ к контейнерам приложения по разным портам из другого Pod внутри кластера

1. Создать Deployment приложения, состоящего из двух контейнеров (nginx и multitool), с количеством реплик 3 шт.
2. Создать Service, который обеспечит доступ внутри кластера до контейнеров приложения из п.1 по порту 9001 — nginx 80, по 9002 — multitool 8080.
3. Создать отдельный Pod с приложением multitool и убедиться с помощью `curl`, что из пода есть доступ до приложения из п.1 по разным портам в разные контейнеры.
4. Продемонстрировать доступ с помощью `curl` по доменному имени сервиса.
5. Предоставить манифесты Deployment и Service в решении, а также скриншоты или вывод команды п.4.


<details>
  <summary>deployment04.yaml</summary>
  
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
  labels:
    app: dual-container-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: dual-container-app
  template:
    metadata:
      labels:
        app: dual-container-app
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.25
        ports:
        - containerPort: 80
      - name: multitool-container
        image: busyboxplus:curl
        ports:
        - containerPort: 8080
        command: ['sh', '-c', 'echo "Hello from multitool!" > /tmp/index.html && httpd -p 8080 -h /tmp && tail -f /dev/null']
```
</details>

<details>
  <summary>service04.yaml</summary>
 
```
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: dual-container-app
  ports:
    - name: nginx-port
      protocol: TCP
      port: 9001
      targetPort: 80
    - name: multitool-port
      protocol: TCP
      port: 9002
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

![01](https://github.com/Myash-New/Kubernetes/blob/main/04/01.jpg)
![02](https://github.com/Myash-New/Kubernetes/blob/main/04/02.jpg)


------

### Задание 2. Создать Service и обеспечить доступ к приложениям снаружи кластера

1. Создать отдельный Service приложения из Задания 1 с возможностью доступа снаружи кластера к nginx, используя тип NodePort.
2. Продемонстрировать доступ с помощью браузера или `curl` с локального компьютера.
3. Предоставить манифест и Service в решении, а также скриншоты или вывод команды п.2.

<details>
  <summary>nodeport-service.yaml</summary>
  
```
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: dual-container-app
  ports:
    - name: web
      protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 32080

 ```

</details>

![03](https://github.com/Myash-New/Kubernetes/blob/main/04/03.jpg)
![04](https://github.com/Myash-New/Kubernetes/blob/main/04/04.jpg)


