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

![01](https://github.com/Myash-New/Kubernetes/blob/main/06/01.jpg)

------

# **Задание 2: Настройка HTTPS с Secrets**  
### **Задача**  
Развернуть приложение с доступом по HTTPS, используя самоподписанный сертификат.

### **Шаги выполнения**  
1. **Сгенерировать SSL-сертификат**
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=myapp.example.com"
```
2. **Создать Secret**
3. **Настроить Ingress**
4. **Проверить HTTPS-доступ**

### **Решение**

<details>
  <summary>secret-tls.yaml</summary>
---
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
  namespace: default
type: kubernetes.io/tls
data:
  tls.crt: |
    LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURHVENDQWdHZ0F3SUJBZ0lVQ056MVBiYnRtcEJJdVVnWm44bmRlK0wwQjQ0d0RRWUpLb1pJaHZjTkFRRUwKQlFBd0hERWFNQmdHQTFVRUF3d1JiWGxoY0hBdVpYaGhiWEJzWlM1amIyMHdIaGNOTWpVd09EQXpNVGswTmpFeApXaGNOTWpZd09EQXpNVGswTmpFeFdqQWNNUm93R0FZRFZRUUREQkZ0ZVdGd2NDNWxlR0Z0Y0d4bExtTnZiVENDCkFTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTkhYTERsOGVnaTNFbllPVkVnbWlCaTUKalhLWU1zcUttV29ZNFkzVitFc2FxMXIzbDA1KyttdEJCeGl4RTlyLzF3OGFNbWlDaWF4SW9LOHp3VFlvbDA3TQpPdGZMdzRjd241bm5wL1htV21KM3I5KzhxUE9kbWs3d0Y4blI0Z0hlYTk5Ujc4d2JDVk1NZExTYnE2dGhIVzZoCkxCdkNNZ0I0aEdRSFB3dU8rZ2lGOTBhd1hzQ2sveWRZbXBCSzFXQlkzNE1JWlBCaG9Ra3dQMHhIVEFGMnVhd0IKSjRjeDZVck9UNW5hR2ZSaTJEeTQra2dCYUxlaE0wZm5NMURxdVhJdFJGektxNXJuOFI0dnFMQkszMmtUVGM3bwpyeWxXWERUcUxqZHlnb3ptT0dzeU9rQm4xellFU1dPZktDSXBhVFhVclE0dFRibTYrNllnZVlKeE16S0IvOE1DCkF3RUFBYU5UTUZFd0hRWURWUjBPQkJZRUZPWjllSW0rWE55UTZHMjNnby9NaDNIK2NHb21NQjhHQTFVZEl3UVkKTUJhQUZPWjllSW0rWE55UTZHMjNnby9NaDNIK2NHb21NQThHQTFVZEV3RUIvd1FGTUFNQkFmOHdEUVlKS29aSQpodmNOQVFFTEJRQURnZ0VCQUEvbW8weEt6QzA2UURJT1NkMlRGUXhXa3RVa1BOMDFYRnU0OUkvT0ZScWdwdUc3CjFBTXJlakp5K3ErNS9UUXVNSlJSWDBRd21zQXUxSDU2dG1SOW1xQWVtZHEzdlBkZ2ljU2JTRzk2NU0xajU0NnEKN3ZOMXBMUHFlNjAvMnptV2YxVmNqNEpaa3p3VW0vbHFuYXMrK2t2RWN5bUxDeitYLy9SK3JKUU9ld2NSZmt0aApYR0JEV2VhUklIbGw3NmFpRGIwU2F3TG9hN0tEZlAvUUp1WDE2KzdWZm50QmwvRi9iOG1LeXhubDRqOXpTZDNRCjg3NHJIOC9IdjhIZFlMY0QyMUl5R1Q1emNIN2s1bU04VC9nakljYnZVc0dYdUNiSWVGblhuWGtmelprd05nejMKRWtjcmlNK3JwdnBzL3B5bC84NDF4T0dLcXBHZ0xlbW03V0NpSWFJPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
  tls.key: |
    LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JSUV2UUlCQURBTkJna3Foa2lHOXcwQkFRRUZBQVNDQktjd2dnU2pBZ0VBQW9JQkFRRFIxeXc1ZkhvSXR4SjIKRGxSSUpvZ1l1WTF5bURMS2lwbHFHT0dOMWZoTEdxdGE5NWRPZnZwclFRY1lzUlBhLzljUEdqSm9nb21zU0tDdgpNOEUyS0pkT3pEclh5OE9ITUorWjU2ZjE1bHBpZDYvZnZLanpuWnBPOEJmSjBlSUIzbXZmVWUvTUd3bFRESFMwCm02dXJZUjF1b1N3YndqSUFlSVJrQno4TGp2b0loZmRHc0Y3QXBQOG5XSnFRU3RWZ1dOK0RDR1R3WWFFSk1EOU0KUjB3QmRybXNBU2VITWVsS3prK1oyaG4wWXRnOHVQcElBV2kzb1ROSDV6TlE2cmx5TFVSY3lxdWE1L0VlTDZpdwpTdDlwRTAzTzZLOHBWbHcwNmk0M2NvS001amhyTWpwQVo5YzJCRWxqbnlnaUtXazExSzBPTFUyNXV2dW1JSG1DCmNUTXlnZi9EQWdNQkFBRUNnZ0VBR3B5YjRvWjR3Tll4ZWRZRFlQVzhyaUMwdHVNUHZlNnAyaHkzQTMwdW4wZTQKa2pGeXpPVCs3bm5jRG9qcjB3cm9iby9oODFocXRxQ3NwWFlvMEt6V2ZqWEVXS01NeWFqNW4xTytVZlJ1dUt6TwpieVRHbGNRQ1VuUjhiOVNLN0dCcXljZzRZeExOWEplVElyeURTWmxvQkNSZDFhb0x4cWVDRXJWMTRnb0xNajFLCi9Yem1uZVNUOUEveG5yUHpPdEx3YjlVaEpBN2VhMjQxYnBxRm54QWdlbjVsRWlza2c0dDhzU3c2MTZXTVMxMDgKU28wenpJSnp6NHNBdm1tTUY4Ty8zcG9VU0pMWG54Vm51U0JaRHRlcWJUeXNyNTUzTmpCbW9jbVNzQ2JJcUtnMgoxRGFRTGplUnBBSWlLQUxTcEhac3p3M0I3cVczU1cvVVVaSmRzWjdPVVFLQmdRRHJTcEtMNGxTKytzNitGQlRZCmxTN0dvWm8zNnFkSUhZRHdjLy9UWkFaamZBV0pQd3NVcGwwd0UrVHJaSTVYT1hXc3lwVmFVNVBrMVNrYkxIRi8Kc1VVRThvd1I3bUdsdUVBd1EranNmUHBPTkJLYk9MRmN0K0ZZWFFGM0dHbDE5YVIyQnNVdmwyMXQxRHFQYU5nTAp3M2dFV3NtcEtjeHVZUzMxQ3hBaXYxdVpEd0tCZ1FEa1R5a0MrZVVyWk9oNDM5WG85RW9NOTY2cUZJaGt0L1hXClEzTGFYdUQ3SHFBSFVVaG5VVXlBMndzc2JoblVTcnlHMlRuQjA5K2c1S2x0bFRLSUFwU2FSS0NmdGg3V1VIeVQKd2pGaFBJZEU1b2hSTkg3Q0hLM3MrQkR1RkQwK0xLdm44UVpMajZYcDRxL0pyRW9kNmxTOEZKOHVwRytSLzJ6WQo1cUlKWGEwbURRS0JnQVNJMEdBdndYQll4d2swdTk0Y3FlVWNFaXZIc3VlWjRmVkFWd3JNMzY2bElqb3Q0OW5ICkJ2NjVNMjB4NStoWWJDTWpXRk9BVHRaWElVNnJ3Wmd6WTJBZ0NJRUQ5Zy9LaURvbDVPUkIyRlVQZmRoTjlHVVUKQ2h5NDFpRmtjQXZjNndsM1FlK1QzSUVFV1FpUWZiRmtWL2pGZ3lObWNkRWl3RTc3b3BqNDFSd1RBb0dBU2dnTgpNV1RjMWVSanFZWlRjN1Y3S1pkSzhPVzFrSXRDVVJjUDhCVmgrS3Zta2xqZUZIcDlSeTlBQVVrMllPdFhGSmJ0CnJwZElaWUNnRytPTVBpUXdFWkg5VDZ5YmRUMG1HRGVaRVlHeUR6cDlxMjlOUng1TG01S1kwc3FIVFZqbzZVM3oKajU3bDJ1Qmh4aEJ5L0I1WEdhSEtPREtqNXdDZlIvb0pRdVk0VmlVQ2dZRUFtR25Ja0VtSFdYQUlKay9adzl4agpsL0NBSXFsUFovNjMrc2FFTFpNMGpjRXg4WjVwNFVoNTAxRUNWd09tdmF1VlpWR0xYZ3JrcVBpQ28zbjVPRHIvCmZwRFhiaE5taVhTSGMvclB1Zndzcjg0eUFudDZHU3krVzJaaVhqZU5wU1NTTU1odUJBZzRXSWxNaklQcnZmQkwKVVpCOGQrRXIvZFBtMkk2MTFyTlZLRWM9Ci0tLS0tRU5EIFBSSVZBVEUgS0VZLS0tLS0K
```
</details>


<details>
  <summary>ingress-tls.yaml</summary>
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: tls-secret
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```
</details>

![02](https://github.com/Myash-New/Kubernetes/blob/main/06/02.jpg)

------

## **Задание 3: Настройка RBAC**  
### **Задача**  
Создать пользователя с ограниченными правами (только просмотр логов и описания подов).

### **Шаги выполнения**  
1. **Включите RBAC в microk8s**
```bash
microk8s enable rbac
```
2. **Создать SSL-сертификат для пользователя**
```bash
openssl genrsa -out developer.key 2048
openssl req -new -key developer.key -out developer.csr -subj "/CN={ИМЯ ПОЛЬЗОВАТЕЛЯ}"
openssl x509 -req -in developer.csr -CA {CA серт вашего кластера} -CAkey {CA ключ вашего кластера} -CAcreateserial -out developer.crt -days 365
```
3. **Создать Role (только просмотр логов и описания подов) и RoleBinding**
4. **Проверить доступ**


### **Решение**

<details>
  <summary>role-pod-reader.yaml</summary>
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log", "pods/status"]
  verbs: ["get", "list"]
```
</details>       

<details>
  <summary>rolebinding-developer.yaml</summary>  
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: developer
  apiGroup: ""
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```
</details> 

![03](https://github.com/Myash-New/Kubernetes/blob/main/06/03.jpg)