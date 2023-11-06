University: [ITMO University](https://itmo.ru/ru/) 
Faculty: [FICT](https://fict.itmo.ru) 
Course: [Introduction to distributed technologies](https://github.com/itmo-ict-faculty/introduction-to-distributed-technologies) 
Year: 2023/2024 
Group: K4113c 
Author: Petrov Aleksandr Denisovich 
Lab: Lab3
Date of create: 06.11.2023 
Date of finished: to be added
___
# Отчёт о лабораторной работе №3

## Развертывание веб сервиса в Minikube, доступ к веб интерфейсу сервиса. Мониторинг сервиса.

- [Описание работы](#описание-работы)
- [Цель работы](#цель-работы)
- [Ход работы](#ход-работы)
  * [Создание configMap:](#создание-configmap)
  * [Создание replicaSet](#создание-replicaset)
  * [Создание Ingress](#создание-ingress)
  * [Проверка сертификата](#проверка-сертификата)
- [Схема организации контейнеров и сервисов](#cхема-организации-контейнеров-и-сервисов)
### Описание работы

В данной лабораторной работе предлагается познакомиться с сертификатами и "секретами" в Minikube, а также правилами безопасного хранения данных в Minikube.
### Цель работы

Познакомиться с сертификатами и "секретами" в Minikube, правилами безопасного хранения данных в Minikube.
### Ход работы

#### Создание configMap
Создадим манифест для объявления configMap, в котором объявим две переменных `user` и `company`. Эти переменные нам потребуется передать в контейнер на следующем этапе:
```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: configmap-demo
data:
  user: "SanchPet"
  company: "Monsters Inc"
```
Применим манифест и создадим configMap, чтобы передавать параметры в контейнеры.
```bash
ubuntu@ubuntu-2204:~/ITMO/lab3$ kubectl apply -f configmap.yml 
configmap/configmap-demo created
```
#### Создание replicaSet
Далее создадим манифест для replicaSet и укажем заданные по заданию параметры. Число реплик - две, образ контейнера - ifilyaninitmo/itdt-contained-frontend:master, порт контейнера - 3000. 
С помощью секции `env` передадим в контейнер переменные окружения, которые подставляются из созданного нами ранее configMap.
```yml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: itdt-contained-frontend
  labels:
    app: front
    tier: itdt-contained-frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      tier: itdt-contained-frontend
  template:
    metadata:
      labels:
        tier: itdt-contained-frontend
    spec:
      containers:
      - name: frontend-container
        image: ifilyaninitmo/itdt-contained-frontend:master
        ports:
        - containerPort: 3000
        env:
        - name: REACT_APP_USERNAME
          valueFrom:
            configMapKeyRef:
              name: configmap-demo
              key: user
        - name: REACT_APP_COMPANY_NAME
          valueFrom:
            configMapKeyRef:
              name: configmap-demo
              key: company
```
Применим манифест и создадим replicaSet:
```bash
ubuntu@ubuntu-2204:~/ITMO/lab3$ kubectl apply -f itdt-replica.yml 
replicaset.apps/itdt-contained-frontend created
```
Проверим работу реплики:
```bash
ubuntu@ubuntu-2204:~/ITMO/lab3$ kubectl expose replicaset itdt-contained-frontend --type=NodePort --port=3000
service/itdt-contained-frontend exposed

ubuntu@ubuntu-2204:~/ITMO/lab3$ kubectl port-forward services/itdt-contained-frontend 3000:3000 
Forwarding from 127.0.0.1:3000 -> 3000
Forwarding from [::1]:3000 -> 3000
```
Перейдём в браузер и убедимся, что все переменные на месте:
![img1](https://github.com/sanchpet/2023_2024-introduction_to_distributed_technologies-k4113c-petrov_a_d/blob/main/lab3/img/Pasted%20image%2020231106203637.png)
#### Создание Ingress
Установим плагин Ingress в minikube:
```bash
ubuntu@ubuntu-2204:~minikube addons enable ingressss
💡  ingress is an addon maintained by Kubernetes. For any concerns contact minikube on GitHub.
You can view the list of minikube maintainers at: https://github.com/kubernetes/minikube/blob/master/OWNERS
    ▪ Using image registry.k8s.io/ingress-nginx/controller:v1.8.1
    ▪ Using image registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20230407
    ▪ Using image registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20230407
🔎  Verifying ingress addon...
🌟  The 'ingress' addon is enabled
```
Сгенерируем TLS сертификат и импортируем его в minikube, добавив также в Ingress:
```bash
ubuntu@ubuntu-2204:~/certs$ openssl req -new -newkey rsa:4096 -x509 -sha256 -days 365 -nodes -out MyCertificate.crt -keyout MyKey.key

ubuntu@ubuntu-2204:~/certs$ kubectl -n kube-system create secret tls mkcert --key MyKey.key --cert MyCertificate.crt
secret/mkcert created

ubuntu@ubuntu-2204:~/certs$ minikube addons configure ingress
-- Enter custom cert (format is "namespace/secret"): kube-system/mkcert
✅  ingress was successfully configured
```
И выполним проверку установки:
```bash
ubuntu@ubuntu-2204:~/certs$ kubectl -n ingress-nginx get deployment ingress-nginx-controller -o yaml | egrep -o "\-\-default-ssl-certificate=kube-system/mkcert"
--default-ssl-certificate=kube-system/mkcert
--default-ssl-certificate=kube-system/mkcert
```
Теперь можно создать манифест для Ingress, указав в нём FQDN и TLS-сертификат. Также указываем сервис, который мы создали для доступа в реплику:
```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: "nginx"
  name: virtual-host-ingress
  namespace: default
spec:
  tls:
    - hosts:
        - "monsters.inc"
    - secretName: mkcert
  rules:
  - host: monsters.inc
    http:
      paths:
      - backend:
          service:
            name: itdt-contained-frontend
            port:
              number: 3000
        path: /
        pathType: ImplementationSpecific
```
После сохранения манифеста создадим Ingress:
```bash
ubuntu@ubuntu-2204:~/certs$ kubectl apply -f ingress.yml 
Warning: annotation "kubernetes.io/ingress.class" is deprecated, please use 'spec.ingressClassName' instead
ingress.networking.k8s.io/virtual-host-ingress created
```
#### Проверка сертификата
Чтобы настроить разрешение нашего FQDN, надо узнать IP, с которого работает Ingress. Это можно сделать командой `minikube ip`, но можно и попросить kubectl описать Ingress:
```bash
ubuntu@ubuntu-2204:~/certs$ kubectl describe ingress virtual-host-ingress 
Name:             virtual-host-ingress
Labels:           <none>
Namespace:        default
Address:          192.168.49.2
Ingress Class:    <none>
Default backend:  <default>
TLS:
  SNI routes monsters.inc
  mkcert terminates 
Rules:
  Host          Path  Backends
  ----          ----  --------
  monsters.inc  
                /   itdt-contained-frontend:443 (10.244.0.34:3000,10.244.0.35:3000)
Annotations:    kubernetes.io/ingress.class: nginx
Events:
  Type    Reason  Age                From                      Message
  ----    ------  ----               ----                      -------
  Normal  Sync    51s (x2 over 77s)  nginx-ingress-controller  Scheduled for sync
```
IP `192.168.49.2` надо присвоить доменному имени `monsters.inc` через `/etc/hosts`:
```bash
ubuntu@ubuntu-2204:~/certs$ cat /etc/hosts
127.0.0.1	localhost
127.0.1.1	ubuntu-2204.linuxvmimages.local	ubuntu-2204

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

192.168.49.2 monsters.inc
```

Теперь, вбивая в Mozilla наше FQDN, мы попадаем в веб-приложение. Подключение при этом производится по протоколу HTTPS:
![img2](https://github.com/sanchpet/2023_2024-introduction_to_distributed_technologies-k4113c-petrov_a_d/blob/main/lab3/img/Pasted%20image%2020231106220034.png)

Убедимся, что выдаётся наш сертификат:

![img3](https://github.com/sanchpet/2023_2024-introduction_to_distributed_technologies-k4113c-petrov_a_d/blob/main/lab3/img/Pasted%20image%2020231106220344.png)

Таким образом, нам удалось настроить Ingress на работу с созданным нами сервисом, используя FQDN и TLS-сертификат. 

### Схема организации контейнеров и сервисов

// TO BE ADDED //

