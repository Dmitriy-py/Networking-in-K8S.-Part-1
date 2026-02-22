# Домашнее задание к занятию «Сетевое взаимодействие в K8S. Часть 1»
## ` Дмитрий Климов `

## Задание 1. Создать Deployment и обеспечить доступ к контейнерам приложения по разным портам из другого Pod внутри кластера
   1. Создать Deployment приложения, состоящего из двух контейнеров (nginx и multitool), с количеством реплик 3 шт.
   2. Создать Service, который обеспечит доступ внутри кластера до контейнеров приложения из п.1 по порту 9001 — nginx 80, по 9002 —          multitool 8080.
   3. Создать отдельный Pod с приложением multitool и убедиться с помощью curl, что из пода есть доступ до приложения из п.1 по разным        портам в разные контейнеры.
   4. Продемонстрировать доступ с помощью curl по доменному имени сервиса.
   5. Предоставить манифесты Deployment и Service в решении, а также скриншоты или вывод команды п.4.

## Ответ:

# Решение Задания 1: Сетевое взаимодействие в K8s

## Описание задачи
1. Создать Deployment для приложения Nginx.
2. Создать Deployment для приложения Multitool.
3. Обеспечить доступ к приложениям через независимые сервисы ClusterIP.
4. Проверить доступность по разным портам:
   - `http://nginx-service:9001`
   - `http://multitool-service:9002`

---

## 1. Манифесты Nginx

**nginx-deployment.yaml**
```yaml
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
      containers:
      - name: nginx-container
        image: nginx:latest
        ports:
        - containerPort: 80
```

**nginx-service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 9001
      targetPort: 80
```

---

## 2. Манифесты Multitool

**multitool-deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multitool-deployment
  labels:
    app: multitool
spec:
  replicas: 1
  selector:
    matchLabels:
      app: multitool
  template:
    metadata:
      labels:
        app: multitool
    spec:
      containers:
      - name: multitool-container
        image: wbitt/network-multitool:latest
        ports:
        - containerPort: 8080
          name: http-multitool
        env:
        - name: HTTP_PORT
          value: "8080"
```

**multitool-service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: multitool-service
spec:
  selector:
    app: multitool
  ports:
    - protocol: TCP
      port: 9002
      targetPort: 8080
```

<img width="1920" height="1080" alt="Снимок экрана (2723)" src="https://github.com/user-attachments/assets/74e896e5-c946-407b-9195-444019b75d1f" />

---

## 3. Проверка сетевой связности

### Состояние подов в кластере:
```bash
cloudshell-user:~$ kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
nginx-deployment-589d57c78b-crgt4       1/1     Running   0          15m
multitool-deployment-5df94f5576-k4kgg   1/1     Running   0          5m
```

### Тестирование доступа (curl):
Запускаем тестовый контейнер и проверяем доступ к обоим сервисам по их внутренним именам:

```bash
kubectl run test-client --image=praqma/network-multitool:latest --rm -it --restart=Never -- /bin/bash

# Проверка Nginx
bash-5.1# curl http://nginx-service:9001
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</html>

# Проверка Multitool
bash-5.1# curl http://multitool-service:9002
WBITT Network MultiTool (with NGINX) - multitool-deployment-5df94f5576-k4kgg - 10.112.128.69 - HTTP: 8080 , HTTPS: 443
```

<img width="1920" height="1080" alt="Снимок экрана (2722)" src="https://github.com/user-attachments/assets/119b0e33-6b7c-4e05-a49a-02547fc90149" />


---

## Задание 2. Создать Service и обеспечить доступ к приложениям снаружи кластера
   1. Создать отдельный Service приложения из Задания 1 с возможностью доступа снаружи кластера к nginx, используя тип NodePort.
   2. Продемонстрировать доступ с помощью браузера или curl с локального компьютера.
   3. Предоставить манифест и Service в решении, а также скриншоты или вывод команды п.2.

## Ответ:

# Решение Задания 2: Доступ к приложению снаружи кластера (NodePort)

## Описание задачи
1. Создать Service типа `NodePort` для приложения Nginx, развернутого в Задании 1.
2. Продемонстрировать доступ к приложению снаружи кластера через внешний IP узла и назначенный порт.

---

## 1. Манифест Service (`nginx-nodeport-service.yaml`)

Для обеспечения доступа извне был создан сервис с типом `NodePort`, сопоставленный с метками пода Nginx.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport-service
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
```

---

## 2. Команды развертывания и получение данных для доступа

Применяем манифест и определяем внешний IP-адрес узла кластера:

```bash
# Применение манифеста
cloudshell-user:~$ kubectl apply -f nginx-nodeport-service.yaml
service/nginx-nodeport-service created

# Проверка порта сервиса
cloudshell-user:~$ kubectl get svc nginx-nodeport-service
NAME                     TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-nodeport-service   NodePort   10.96.160.6    <none>        80:30080/TCP   20s

# Определение внешнего IP узла
cloudshell-user:~$ kubectl get nodes -o wide
NAME                        STATUS   ROLES    AGE    INTERNAL-IP   EXTERNAL-IP       ...
cl1jfjbjjp9qablofb5-onef    Ready    <none>   6h29m  10.127.0.4    158.160.212.128   ...
```

<img width="1920" height="1080" alt="Снимок экрана (2725)" src="https://github.com/user-attachments/assets/953b1003-e912-4240-b254-87166c4b9051" />


---

## 3. Проверка доступа снаружи кластера

Проверка выполнялась с локальной машины (VirtualBox) с использованием утилиты `curl` и веб-браузера.

### 3.1. Проверка через curl:
```bash
vm1@vm1-VirtualBox:~$ curl http://158.160.212.128:30080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and working.</p>
...
</body>
</html>
```

### 3.2. Проверка через веб-браузер:
Доступ был получен по адресу: `http://158.160.212.128:30080`

---
<img width="1920" height="1080" alt="Снимок экрана (2726)" src="https://github.com/user-attachments/assets/49a4bf8e-492a-4785-aa17-580696fc880d" />

<img width="1920" height="1080" alt="Снимок экрана (2727)" src="https://github.com/user-attachments/assets/be648b98-03e7-457c-96f0-166046c60c82" />















