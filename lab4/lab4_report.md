University: [ITMO University](https://itmo.ru/ru/) 
Faculty: [FICT](https://fict.itmo.ru) 
Course: [Introduction to distributed technologies](https://github.com/itmo-ict-faculty/introduction-to-distributed-technologies) 
Year: 2023/2024 
Group: K4113c 
Author: Petrov Aleksandr Denisovich 
Lab: Lab4
Date of create: 07.11.2023 
Date of finished: to be added
___
# Отчёт о лабораторной работе №4

## Сети связи в Minikube, CNI и CoreDNS

- [Описание работы](#описание-работы)
- [Цель работы](#цель-работы)
- [Ход работы](#ход-работы)
  * [Включение Calico в Minikube и запуск кластера на 2 ноды:](#включение-calico-в-minikube-и-запуск-кластера-на-2-ноды)
  * [Настройка IPAM](#настройка-ipam)
  * [Проверка правил выдачи IP-адресов](#проверка-правил-выдачи-ip-адресов)
  * [Связность подов внутри сети](#связность-подов-внутри-сети)
- [Схема организации контейнеров и сервисов](#cхема-организации-контейнеров-и-сервисов)
### Описание работы
Данная лабораторная работа является четвёртой и последней в курсе. В её рамках предлагается познакомиться с сетями связи в Minikube. Особенность Kubernetes заключается в том, что у него одновременно работают `underlay` и `overlay` сети, а управление может быть организованно различными CNI.

### Цель работы

Познакомиться с CNI Calico и функцией `IPAM Plugin`, изучить особенности работы CNI и CoreDNS.

### Ход работы

#### Включение Calico в Minikube и запуск кластера на 2 ноды
Включим minikube, установив в него плагин CNI Calico и передав опцию для создания двух нод в кластере:
```bash
ubuntu@ubuntu-2204:~/ITMO$ minikube start --network-plugin=cni --cni=calico --nodes 2 -p multinode-demo
😄  [multinode-demo] minikube v1.31.2 on Ubuntu 22.04 (vbox/amd64)
✨  Automatically selected the docker driver. Other choices: ssh, none
❗  With --network-plugin=cni, you will need to provide your own CNI. See --cni flag as a user-friendly alternative
📌  Using Docker driver with root privileges
👍  Starting control plane node multinode-demo in cluster multinode-demo
🚜  Pulling base image ...
🔥  Creating docker container (CPUs=2, Memory=2200MB) ...
🐳  Preparing Kubernetes v1.27.4 on Docker 24.0.4 ...
    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔗  Configuring Calico (Container Networking Interface) ...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🔎  Verifying Kubernetes components...
🌟  Enabled addons: storage-provisioner, default-storageclass

👍  Starting worker node multinode-demo-m02 in cluster multinode-demo
🚜  Pulling base image ...
🔥  Creating docker container (CPUs=2, Memory=2200MB) ...
🌐  Found network options:
    ▪ NO_PROXY=192.168.58.2
🐳  Preparing Kubernetes v1.27.4 on Docker 24.0.4 ...
    ▪ env NO_PROXY=192.168.58.2
🔎  Verifying Kubernetes components...
🏄  Done! kubectl is now configured to use "multinode-demo" cluster and "default" namespace by default
```
Убедимся в том, что настройки применились корректно. Проверим ноды:
```bash
ubuntu@ubuntu-2204:~/ITMO$ kubectl get nodes
NAME                 STATUS   ROLES           AGE    VERSION
multinode-demo       Ready    control-plane   4m9s   v1.27.4
multinode-demo-m02   Ready    <none>          3m7s   v1.27.4
```
И сверимся, что Calico был добавлен в кластер:
```bash
ubuntu@ubuntu-2204:~/ITMO$ kubectl get pods -l k8s-app=calico-node -A
NAMESPACE     NAME                READY   STATUS    RESTARTS   AGE
kube-system   calico-node-cqb7r   1/1     Running   0          4m36s
kube-system   calico-node-w85xx   1/1     Running   0          5m22s
```

#### Настройка IPAM
Присвоим каждой ноде новый `label`, который будет описывать, в какой стойке датацентра работает каждая нода:
```bash
ubuntu@ubuntu-2204:~/ITMO$ kubectl label nodes multinode-demo rack=c01
node/multinode-demo labeled

ubuntu@ubuntu-2204:~/ITMO$ kubectl label nodes multinode-demo-m02 rack=c02
node/multinode-demo-m02 labeled

ubuntu@ubuntu-2204:~/ITMO$ kubectl describe node multinode-demo | grep rack
                    rack=c01
                    
ubuntu@ubuntu-2204:~/ITMO$ kubectl describe node multinode-demo-m02 | grep rack
                    rack=c02
```

Установим `calicoctl` и проверим его работу:
```bash
ubuntu@ubuntu-2204:~/ITMO$ cd /usr/local/bin

ubuntu@ubuntu-2204:/usr/local/bin$ sudo curl -L https://github.com/projectcalico/calico/releases/download/v3.26.3/calicoctl-linux-amd64 -o calicoctl
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100 62.0M  100 62.0M    0     0  18.9M      0  0:00:03  0:00:03 --:--:-- 23.9M

ubuntu@ubuntu-2204:/usr/local/bin$ sudo chmod +x ./calicoctl

ubuntu@ubuntu-2204:/usr/local/bin$ calicoctl 
Usage:
  calicoctl [options] <command> [<args>...]
Invalid option: ''. Use flag '--help' to read about a specific subcommand.
```

Убедимся, что Calico выдал диапазон по умолчанию:
```bash
ubuntu@ubuntu-2204:~/ITMO/lab4$ calicoctl get ippool -o wide --allow-version-mismatch
NAME                  CIDR            NAT    IPIPMODE   VXLANMODE   DISABLED   DISABLEBGPEXPORT   SELECTOR   
default-ipv4-ippool   10.244.0.0/16   true   Always     Never       false      false              all()  
```

Удалим диапазон:
```bash
ubuntu@ubuntu-2204:~/ITMO/lab4$ calicoctl delete ippools default-ipv4-ippool --allow-version-mismatch
Successfully deleted 1 'IPPool' resource(s)
```

Создадим манифест для выдачи диапазонов в соответствии с распределением нод по стойкам:
```yml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: rack-c01-pool
spec:
  cidr: 10.244.1.0/24
  ipipMode: Always
  natOutgoing: true
  nodeSelector: rack == "c01"

---

apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: rack-c02-pool
spec:
  cidr: 10.244.2.0/24
  ipipMode: Always
  natOutgoing: true
  nodeSelector: rack == "c02"
```

Применим манифест и сверимся с новыми диапазонами:
```bash
ubuntu@ubuntu-2204:~/ITMO/lab4$ calicoctl apply -f calico.yml --allow-version-mismatch
Successfully applied 2 'IPPool' resource(s)

ubuntu@ubuntu-2204:~/ITMO/lab4$ calicoctl get ippool -o wide --allow-version-mismatch
NAME            CIDR            NAT    IPIPMODE   VXLANMODE   DISABLED   DISABLEBGPEXPORT   SELECTOR        
rack-c01-pool   10.244.1.0/24   true   Always     Never       false      false              rack == "c01"   
rack-c02-pool   10.244.2.0/24   true   Always     Never       false      false              rack == "c02"
```

#### Проверка правил выдачи IP-адресов
По заданию создадим deployment, взяв уже знакомый нам контейнер и передав в него переменные окружения. Применим configMap из 3-й лабораторной работы и манифест deployment'а из 2-й работы, отредактировав его для забора значений переменных из configMap:
```bash
ubuntu@ubuntu-2204:~/ITMO/lab4$ cp ../lab3/configmap.yml . 
ubuntu@ubuntu-2204:~/ITMO/lab4$ cat configmap.yml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: configmap-demo
data:
  user: "SanchPet"
  company: "Monsters Inc"
```
```bash
ubuntu@ubuntu-2204:~/ITMO/lab4$ cp ../lab2/itdt-deployment.yaml .

ubuntu@ubuntu-2204:~/ITMO/lab4$ nano itdt-deployment.yaml

ubuntu@ubuntu-2204:~/ITMO/lab4$ cat itdt-deployment.yaml 
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
Запустим configMap и deployment:
```bash
ubuntu@ubuntu-2204:~/ITMO/lab4$ kubectl apply -f configmap.yml 
configmap/configmap-demo created

ubuntu@ubuntu-2204:~/ITMO/lab4$ kubectl apply -f itdt-deployment.yaml 
deployment.apps/itdt-contained-frontend-deployment created
```
Создадим сервис для deployment'а и сразу пробросим порт:
```bash
ubuntu@ubuntu-2204:~/ITMO/lab4$ kubectl expose deployment itdt-contained-frontend-deployment  --type=NodePort --port=3000
service/itdt-contained-frontend-deployment exposed

ubuntu@ubuntu-2204:~/ITMO/lab4$ kubectl port-forward services/itdt-contained-frontend-deployment 3000:3000 
Forwarding from 127.0.0.1:3000 -> 3000
Forwarding from [::1]:3000 -> 3000
```
Проверим, что будет выдавать веб-приложение:

![img1](https://github.com/sanchpet/2023_2024-introduction_to_distributed_technologies-k4113c-petrov_a_d/blob/main/lab4/img/Pasted%20image%2020231107160912.png)

Браузер всегда обслуживает подключение на один под. Если пересоздавать deployment через `kubectl delete` / `kubectl apply`, можно видеть, что IP присваиваются разные, но строго из описанных нами диапазонов:

![img2](https://github.com/sanchpet/2023_2024-introduction_to_distributed_technologies-k4113c-petrov_a_d/blob/main/lab4/img/Pasted%20image%2020231107162025.png)

![img3](https://github.com/sanchpet/2023_2024-introduction_to_distributed_technologies-k4113c-petrov_a_d/blob/main/lab4/img/Pasted%20image%2020231107162153.png)

![img4](https://github.com/sanchpet/2023_2024-introduction_to_distributed_technologies-k4113c-petrov_a_d/blob/main/lab4/img/Pasted%20image%2020231107164817.png)

И так далее. Естественно, подсеть на поде выбирается в соответствии с нодой, на которой размещён под. 

#### Связность подов внутри сети
В очередной раз пересоздадим deployment и сверимся, что поды получили IP из требуемых диапазонов:
```bash
ubuntu@ubuntu-2204:~/ITMO/lab4$ kubectl describe pods | grep IP
                  cni.projectcalico.org/podIP: 10.244.1.68/32
                  cni.projectcalico.org/podIPs: 10.244.1.68/32
IP:               10.244.1.68
IPs:
  IP:           10.244.1.68
                  cni.projectcalico.org/podIP: 10.244.2.195/32
                  cni.projectcalico.org/podIPs: 10.244.2.195/32
IP:               10.244.2.195
```

Зная IP-адреса, мы можем получить доступ к подам по FQDN, если будем знать FQDN обслуживающего их сервиса. Ещё раз проверим имя сервиса:
```bash
ubuntu@ubuntu-2204:~/ITMO/lab4$ kubectl get service
NAME                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
itdt-contained-frontend-deployment   NodePort    10.107.152.191   <none>        3000:30511/TCP   67m
```

Войдём в один из подов и узнаем FQDN нашего сервиса. 
```bash
buntu@ubuntu-2204:~/ITMO/lab4$ kubectl exec -ti pods/itdt-contained-frontend-deployment-7686fd9f98-bzvqp sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
/frontend # nslookup itdt-contained-frontend-deployment
Server:		10.96.0.10
Address:	10.96.0.10:53


** server can't find itdt-contained-frontend-deployment.cluster.local: NXDOMAIN

Name:	itdt-contained-frontend-deployment.default.svc.cluster.local
Address: 10.107.152.191
```
Зная, что FQDN сервиса = `itdt-contained-frontend-deployment.default.svc.cluster.local`, мы можем пинговать поды сервиса, как его поддомены:
```bash
/frontend # ping 10-244-1-68.itdt-contained-frontend-deployment.default.svc.cluster.local
PING 10-244-1-68.itdt-contained-frontend-deployment.default.svc.cluster.local (10.244.1.68): 56 data bytes
64 bytes from 10.244.1.68: seq=0 ttl=64 time=0.053 ms
64 bytes from 10.244.1.68: seq=1 ttl=64 time=0.045 ms
64 bytes from 10.244.1.68: seq=2 ttl=64 time=0.036 ms
64 bytes from 10.244.1.68: seq=3 ttl=64 time=0.053 ms
64 bytes from 10.244.1.68: seq=4 ttl=64 time=0.041 ms
^C
--- 10-244-1-68.itdt-contained-frontend-deployment.default.svc.cluster.local ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max = 0.036/0.045/0.053 ms
/frontend # ping 10-244-2-195.itdt-contained-frontend-deployment.default.svc.cluster.local
PING 10-244-2-195.itdt-contained-frontend-deployment.default.svc.cluster.local (10.244.2.195): 56 data bytes
64 bytes from 10.244.2.195: seq=0 ttl=62 time=1.018 ms
64 bytes from 10.244.2.195: seq=1 ttl=62 time=0.310 ms
64 bytes from 10.244.2.195: seq=2 ttl=62 time=0.136 ms
64 bytes from 10.244.2.195: seq=3 ttl=62 time=0.198 ms
64 bytes from 10.244.2.195: seq=4 ttl=62 time=0.165 ms
^C
--- 10-244-2-195.itdt-contained-frontend-deployment.default.svc.cluster.local ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max = 0.136/0.365/1.018 ms
```

Таким образом, мы убедились, что в нашем деплойменте поды из разных нод корректно получили IP-адреса от Calico и доступны друг другу по сети. 
### Схема организации контейнеров и сервисов

![img5](https://github.com/sanchpet/2023_2024-introduction_to_distributed_technologies-k4113c-petrov_a_d/blob/main/lab4/img/Pasted%20image%2020231107174247.png)

