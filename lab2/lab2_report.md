University: [ITMO University](https://itmo.ru/ru/) 
Faculty: [FICT](https://fict.itmo.ru) 
Course: [Introduction to distributed technologies](https://github.com/itmo-ict-faculty/introduction-to-distributed-technologies) 
Year: 2023/2024 
Group: K4113c 
Author: Petrov Aleksandr Denisovich 
Lab: Lab2 
Date of create: 21.10.2023 
Date of finished: to be added
___
# Отчёт о лабораторной работе №2

## Развертывание веб сервиса в Minikube, доступ к веб интерфейсу сервиса. Мониторинг сервиса.

- [Описание работы](#описание-работы)
- [Цель работы](#цель-работы)
- [Ход работы](#ход-работы)
  * [Старт minikube:](#cтарт-minikube)
  * [Создание и запуск Deployment](#создание-и-запуск-deployment)
  * [Создание сервиса](#создание-сервиса)
  * [Проброс портов и подключение к контейнерам через браузер](#проброс-портов-и-подключение-к-контейнерам-через-браузер)
  * [Проверка поведения контейнеров](#проверка-поведения-контейнеров)
- [Схема организации контейнеров и сервисов](#cхема-организации-контейнеров-и-сервисов)
- [Вопросы к работе](#вопросы-к-работе)
### Описание работы
Это вторая лабораторная работа курса, в ходе которой предлагается ознакомиться с типами "контроллеров" развертывания контейнеров, ознакомится с сетевыми сервисами и развернуть свое веб приложение.

### Цель работы

Разработать манифест для запуска Deployment и запустить первый Deployment. Потренироваться в создании сервисов для доступа к Deployment или отдельным его подам. Проверить передачу параметров окружения в контейнер через манифест. 
### Ход работы

#### Старт minikube
Запустим minikube командой `minkube start`
```bash
ubuntu@ubuntu-2204:~/ITMO/lab2$ minikube start
😄  minikube v1.31.2 on Ubuntu 22.04 (vbox/amd64)
✨  Using the docker driver based on existing profile
👍  Starting control plane node minikube in cluster minikube
🚜  Pulling base image ...
🏃  Updating the running docker "minikube" container ...
🐳  Preparing Kubernetes v1.27.4 on Docker 24.0.4 ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: default-storageclass, storage-provisioner
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

#### Создание и запуск Deployment
Создадим манифест деплоймента `itdt-deployment.yaml`:
```bash
ubuntu@ubuntu-2204:~/ITMO/lab2$ nano itdt-deployment.yaml
```

Примерное содержимое манифеста по заданию представлено ниже. Основные моменты:
- Подтягиваем образ `ifilyaninitmo/itdt-contained-frontend` по заданию  (не забыть указать версию `:master` из manpage со страницы с образом)
- Укажем нужное число реплик: `replicas: 2`
- Передадим нужные переменные окружения в контейнер с помощью секции `env:`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: itdt-contained-frontend-deployment
  labels:
    app: itdt-contained-frontend-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: itdt-contained-frontend-deployment
  template:
    metadata:
      labels:
        app: itdt-contained-frontend-deployment
    spec:
      containers:
      - name: itdt-contained-frontend-deployment
        image: ifilyaninitmo/itdt-contained-frontend:master
        env:
        - name: REACT_APP_USERNAME
          value: "SanchPet"
        - name: REACT_APP_COMPANY_NAME
          value: "SanchPet Corporation"
        ports:
        - containerPort: 3000
```

Запустим наш Deployment с помощью `kubectl`:
```bash
ubuntu@ubuntu-2204:~/ITMO/lab2$ kubectl apply -f itdt-deployment.yaml
deployment.apps/itdt-contained-frontend-deployment created
```
Проверим статус Deployment:
```bash
ubuntu@ubuntu-2204:~/ITMO/lab2$ kubectl get deployments.apps 
NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
itdt-contained-frontend-deployment   2/2     2            2           45s
```

> Интересный момент. Если при вызове `kubectl get deployments.apps` мы получаем в выводе `READY 0/2`, скорее всего, Deployment ещё в процессе создания. Проверить статус создания можно командой `kubectl rollout status itdt-contained-frontend-deployment`. Если что-то пошло не так, на помощь приходит просмотр логов: `kubectl logs deployments/itdt-contained-frontend-deployment`. Так, например, первоначально я забыл указать версию образа `:master`, и просмотр логов помог мне увидеть, что в процессе запуска Deployment возникали ошибки при подтягивании образа контейнера.

Мы убедились, что реплик действительно две. Можно также глянуть отдельно запущенные поды:
```bash
ubuntu@ubuntu-2204:~/ITMO/lab2$ kubectl get pods
NAME                                                  READY   STATUS    RESTARTS      AGE
itdt-contained-frontend-deployment-68f45fd5d8-f5pfq   1/1     Running   0             4m22s
itdt-contained-frontend-deployment-68f45fd5d8-q4z45   1/1     Running   0             4m22s
```
Кстати, в тех же логах можно проверить, на каком порте стартовал сервис (если почему-то вы не доверяете manpage в страничке образа):
```bash
ubuntu@ubuntu-2204:~/ITMO/lab2$ kubectl logs deployments/itdt-contained-frontend-deployment 
Found 2 pods, using pod/itdt-contained-frontend-deployment-68f45fd5d8-f5pfq
Builing frontend
Browserslist: caniuse-lite is outdated. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
Browserslist: caniuse-lite is outdated. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
build finished
Server started on port 3000
```
#### Создание сервиса
Поскольку я ощущаю себя полным нубом по части создания манифестов, возьму пример создания сервиса из лабораторной работы №1, подставлю в него `deployment` и проверю вывод с помощью опций `--dry-run=client -o yaml`:
```bash
ubuntu@ubuntu-2204:~/ITMO/lab2$ kubectl -- expose deployment itdt-contained-frontend-deployment --type=NodePort --port=3000 --dry-run=client -o yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: itdt-contained-frontend-deployment
  name: itdt-contained-frontend-deployment
spec:
  ports:
  - port: 3000
    protocol: TCP
    targetPort: 3000
  selector:
    app: itdt-contained-frontend-deployment
  type: NodePort
status:
  loadBalancer: {}
```

После чего создам сам сервис:
```bash
ubuntu@ubuntu-2204:~/ITMO/lab2$ minikube kubectl -- expose deployment itdt-contained-frontend-deployment --type=NodePort --port=3000
service/itdt-contained-frontend-deployment exposed
```
Проверю сервис:
```bash
ubuntu@ubuntu-2204:~/ITMO/lab2$ kubectl get services
NAME                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
itdt-contained-frontend-deployment   NodePort    10.104.249.183   <none>        3000:32609/TCP   52s
kubernetes                           ClusterIP   10.96.0.1        <none>        443/TCP          16d
```
#### Проброс портов и подключение к контейнерам через браузер
Выполню проброс портов в терминале и отправлю процесс в фон (также отключу потоки вывода, чтобы отладочная информация не мешалась в терминале)
```bash
ubuntu@ubuntu-2204:~/ITMO/lab2$ kubectl -- port-forward service/itdt-contained-frontend-deployment 3000:3000 1>/dev/null 2>&1 &
[1] 257201
```

Подключимся к контейнерам с браузера:

![img1](https://github.com/sanchpet/2023_2024-introduction_to_distributed_technologies-k4113c-petrov_a_d/blob/main/lab2/img/Pasted%20image%2020231021112327.png)

#### Проверка поведения контейнеров
При подключении к контейнерам нас могут интересовать следующие вещи:
- Как ведут себя переменные `REACT_APP_USERNAME`, `REACT_APP_COMPANY_NAME`, которые мы передали в образ
- Как ведет себя переменная `Container name`, отображающая имя контейнера
- Что сохраняется в логе контейнеров

При перезагрузке страницы переменные `REACT_APP_USERNAME`, `REACT_APP_COMPANY_NAME` и `Container name` не меняются. Port-forward к сервису упорно оставляет соединение только с одним из подов. Проверим логи:
```bash
ubuntu@ubuntu-2204:~/ITMO/lab2$ kubectl logs pods/itdt-contained-frontend-deployment-68f45fd5d8-f5pfq 
Builing frontend
Browserslist: caniuse-lite is outdated. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
Browserslist: caniuse-lite is outdated. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
build finished
Server started on port 3000
ubuntu@ubuntu-2204:~/ITMO/lab2$ kubectl logs pods/itdt-contained-frontend-deployment-68f45fd5d8-q4z45
Builing frontend
Browserslist: caniuse-lite is outdated. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
Browserslist: caniuse-lite is outdated. Please run:
  npx update-browserslist-db@latest
  Why you should do it regularly: https://github.com/browserslist/update-db#readme
build finished
Server started on port 3000
```
Логи такие же, как и при создании деплоймента. 
Попробуем создать новый сервис для доступа к конкретным подам и проверим, будут ли меняться переменные в данном случае.

Создам дополнительные сервисы, которые определю отдельно на каждый под:
```bash
ubuntu@ubuntu-2204:~/ITMO/lab2$ minikube kubectl -- expose pod itdt-contained-frontend-deployment-68f45fd5d8-f5pfq --target-port=3000 --port=3001 --type=NodePort
service/itdt-contained-frontend-deployment-68f45fd5d8-f5pfq exposed
ubuntu@ubuntu-2204:~/ITMO/lab2$ minikube kubectl -- expose pod itdt-contained-frontend-deployment-68f45fd5d8-q4z45 --target-port=3000 --port=3002 --type=NodePort
service/itdt-contained-frontend-deployment-68f45fd5d8-q4z45 exposed
ubuntu@ubuntu-2204:~/ITMO/lab2$ kubectl get services
NAME                                                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
itdt-contained-frontend-deployment                    NodePort    10.104.249.183   <none>        3000:32609/TCP   70m
itdt-contained-frontend-deployment-68f45fd5d8-f5pfq   NodePort    10.99.99.183     <none>        3001:30912/TCP   20s
itdt-contained-frontend-deployment-68f45fd5d8-q4z45   NodePort    10.111.207.189   <none>        3002:31718/TCP   7s
kubernetes                                            ClusterIP   10.96.0.1        <none>        443/TCP          17d
ubuntu@ubuntu-2204:~/ITMO/lab2$ minikube kubectl -- port-forward service/itdt-contained-frontend-deployment-68f45fd5d8-q4z45 3002:3002 1>/dev/null 2>&1 &
[2] 320615
ubuntu@ubuntu-2204:~/ITMO/lab2$ minikube kubectl -- port-forward service/itdt-contained-frontend-deployment-68f45fd5d8-f5pfq 3001:3001 1>/dev/null 2>&1 &
[3] 320806
```

Однако, даже создав дополнительный сервис для пода `itdt-contained-frontend-deployment-68f45fd5d8-q4z45`, при проверке содержимого в браузере я получаю имя первого контейнера:
![img2](https://github.com/sanchpet/2023_2024-introduction_to_distributed_technologies-k4113c-petrov_a_d/blob/main/lab2/img/Pasted%20image%2020231021123223.png)

### Схема организации контейнеров и сервисов

![img3](https://github.com/sanchpet/2023_2024-introduction_to_distributed_technologies-k4113c-petrov_a_d/blob/main/lab2/img/Pasted%20image%2020231021130232.png)
### Вопросы к работе

1. Меняются ли переменные `REACT_APP_USERNAME`, `REACT_APP_COMPANY_NAME`?

Нет, поскольку они передаются в образ для реплики и совпадают для обоих подов.

2. Меняется ли переменная `Container name`?

Нелогично, но не меняется. Даже при развертывании deployment и создания сервиса на второй под реплики, при переходе на него всё равно мы получаем ID контейнера №1 в ReplicaSet. Таким же образом и не меняется IP-адрес.

3. Содержимое логов контейнеров?

Содержимое логов можно проверить командой `kubectl logs pods/<pod_id>`, вывод логов приводился в пункте работы "Проверка поведения контейнеров", дублировать его не вижу смысла. 
