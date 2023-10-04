University: [ITMO University](https://itmo.ru/ru/) 
Faculty: [FICT](https://fict.itmo.ru) 
Course: [Introduction to distributed technologies](https://github.com/itmo-ict-faculty/introduction-to-distributed-technologies) 
Year: 2023/2024 
Group: K4113c 
Author: Petrov Aleksandr Denisovich 
Lab: Lab1 
Date of create: 03.10.2023 
Date of finished: to be added
___

# Отчёт о лабораторной работе №1

## Установка Docker и Minikube, мой первый манифест.

- [Описание работы](#описание-работы)
- [Цель работы](#цель-работы)
- [Ход работы](#ход-работы)
  * [Запуск minikube:](#запуск-minikube)
  * [Запуск контейнера vault](#запуск-контейнера-vault)
  * [Сервис для доступа к контейнеру](#сервис-для-доступа-к-контейнеру)
  * [Поиск токена](#поиск-токена)
  * [Остановка кластера](#остановка-кластера)
- [Вопросы к работе](#вопросы-к-работе)
  * [1. Что сейчас произошло и что сделали команды указанные ранее?](#1-что-сейчас-произошло-и-что-сделали-команды-указанные-ранее)
  * [2. Где взять токен для входа в Vault?](#2-где-взять-токен-для-входа-в-vault)

### Описание работы
Это первая лабораторная работа курса, в ходе которой предлагается установить и запустить Minikube с использованием Docker в качестве драйвера и развернуть первый под. 
### Цель работы
Ознакомиться с Docker и Minikube, создать манифест для запуска пода, запустить и протестировать под. 
### Ход работы
Для проведения работы установлен образ виртуальной машины Linux Ubuntu 22.04 для VirtualBox. Параметры ВМ выставлены соответственно требованиям minikube.
#### Запуск minikube:
```
$ minikube start
😄  minikube v1.31.2 on Ubuntu 22.04 (vbox/amd64)
✨  Automatically selected the docker driver. Other choices: none, ssh
📌  Using Docker driver with root privileges
👍  Starting control plane node minikube in cluster minikube
🚜  Pulling base image ...
🔥  Creating docker container (CPUs=2, Memory=2200MB) ...
🐳  Preparing Kubernetes v1.27.4 on Docker 24.0.4 ...
    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔗  Configuring bridge CNI (Container Networking Interface) ...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🔎  Verifying Kubernetes components...
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```
Дополню, что на машину был также отдельно установлен `kubectl` для более удобного управления контейнерами. 
#### Запуск контейнера vault
Для запуска пода потребуется манифест пода .yaml. Поскольку я являюсь полным нубом в Docker и k8s и курс на edX только начал проходить, то на помощь оперативно пришла [статья на Хабре](https://habr.com/ru/articles/589415/) и шаблон манифеста из статьи. Создан `vault.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vault
  labels:
    app: vault
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vault
  template:
    metadata:
      labels:
        app: vault
    spec:
      containers:
      - name: vault
        image: vault:1.13.3
        ports:
        - containerPort: 8200
```
Также для красоты по совету из статьи я создал отдельный namespace для лабораторной и запустил контейнер в нём. Создал `lab1.yaml`:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lab1
```
Затем применил манифест пространства имён 
```bash
$ kubectl apply -f lab1.yaml
```
И запустил в нём контейнер
```bash
$ kubectl apply -f vault.yaml --namespace lab1
deployment.apps/vault created
```
Проверка статуса контейнера:
```bash
$ kubectl get pods -n lab1
NAME                     READY   STATUS    RESTARTS   AGE
vault-5b56bbdcc8-fdrc4   1/1     Running   0          18s
```
#### Сервис для доступа к контейнеру
Создадим сервис для доступа к контейнеру, используя команду из руководства к выполнению работы:
```bash
$ minikube kubectl -- expose pod vault-5b56bbdcc8-fdrc4 --type=NodePort --port=8200 --namespace=lab1
service/vault-5b56bbdcc8-fdrc4 exposed
```
Прокинем порт 8200 в контейнер командой из руководства:
```bash
$ minikube kubectl -- port-forward service/vault-5b56bbdcc8-fdrc4 8200:8200 --namespace=lab1
```
На скрине ниже приведён пример доступа к контейнеру через веб-браузер:
![img1](https://github.com/sanchpet/2023_2024-introduction_to_distributed_technologies-k4113c-petrov_a_d/blob/main/lab1/img/Pasted%20image%2020231004123309.png)
#### Поиск токена
Отправим процесс проброса сервиса в background (`^Z`, `bg`) и проверим логи пода:
```bash
$ kubectl logs vault-5b56bbdcc8-fdrc4 --namespace=lab1
```
В конце лога видно сгенерированный токен:
```
The unseal key and root token are displayed below in case you want to
seal/unseal the Vault or re-authenticate.

Unseal Key: ccGwtjpshMkrcd7x7pbKZr+Q2KreLpO3x1O6wFfrvf8=
Root Token: hvs.F0BQOHL47ERZthKWRYD4Fna0
```
Авторизуемся в Vault:
![img2](https://github.com/sanchpet/2023_2024-introduction_to_distributed_technologies-k4113c-petrov_a_d/blob/main/lab1/img/Pasted%20image%2020231004124532.png)
#### Остановка кластера
Приостановим процесс проброса порта на сервис и завершим работу кластера:
```bash
$ jobs
[1]+  Running                 minikube kubectl -- port-forward service/vault-5b56bbdcc8-fdrc4 8200:8200 --namespace=lab1 &
$ kill %1
$ minikube stop
✋  Stopping node "minikube"  ...
🛑  Powering off "minikube" via SSH ...
🛑  1 node stopped.
```

### Схема организации контейнеров и сервисов

![[Pasted image 20231004155856.png]]

### Вопросы к работе

#### 1. Что сейчас произошло и что сделали команды указанные ранее?
Был запущен контейнер с сервисом Vault - утилитой для безопасного управления секретов. Первоначально с помощью указанных выше команд был запущен контейнер внутри Minikube. Для работы контейнеру был указан порт 8200, а впоследствии был создан дополнительный уровень абстрации - сервис для доступа к контейнеру. Сервис открывает доступ на порт контейнера. Последней командой был настроен проброс порта 8200 на localhost в порт 8200 на контейнере. 
#### 2. Где взять токен для входа в Vault?
Как было продемонстрировано выше, токен можно узнать из лога пода.
