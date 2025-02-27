# Основы DevOps и DataOps | ЛБ 3 | Kubernetes

**Дубровин Е.Н.**

МГТУ им. Н.Э. Баумана, 2024 г.

---

[TOC]

---

## Цель и задачи лабораторной работы

Цель лабораторной работы: Изучить основные концепции работы с Kubernetes на примере Minikube

Задачи лабораторной работы:

1. Установить и запустить Minikube

2. Запустить простое приложение в формате Pod и Deployment

3. Развернуть БД при помощи StatefullSet

4. Организовать доступ к приложению при помощи Ingress с самоподписанным сертификатом

## Часть 1 | Установка и знакомство с Minikube

Minikube является средством локального развёртывания Kubernetes (k8s), предназначеное для установки на индивидуальный компьютер разработчика.

Minikube вводит меньше ограничений для развёртывания в отличие от классического Kubernetes.

Minikube запускается как отдельное приложение, в качестве виртуальной машины, Docker образа или непосредственного приложения ОС.

Все требуемые для отдельной установки модули Kubernetes устанавливаются в рамках этого приложения на одной машине:

* Control Plane - kube-api, etcd, schelduer, controller-manager
* Node - kubelet, k-proxy
* CRI - CRI-O, containerd (по-умолчанию), docker
* CSI - Local или установленный дополнительно
* CNI - Kindnet - предоставляет сетевые сервисы LoadBalancer, но не содержит возможности настройки сетевых политик (ресурс NetworkPolicy) - CNI может быть изменён
* Прочие приложения, например, Ingress устанавливаются при помощи утилиты Minikube

Minikube реализует в полном объёме API Kubernetes.

### Подготовка

Для работы в рамках лабораторной работы потребуется склонировать репозиторий:

```bash
git clone git@github.com:twobrowin-study/devopsit-lab-3-kubernetes.git
```

📋 Если на рабочей машине не установлен Docker, следует воспользоваться инструкцией, [приведённой на странице ЛБ2](https://github.com/twobrowin-study/devopsit-lab-2-docker#%D1%87%D0%B0%D1%81%D1%82%D1%8C-1--%D1%83%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B0-docker)

### Установка Kubectl

Для управления Kubernetes (в нашем случае, Minikube), необходимо предварительно установить утилиту kubectl.

При доступе из сети Интернет следует скачать последнюю версию при помощи команды:

```bash
curl -LO https://dl.k8s.io/release/`curl -LS https://dl.k8s.io/release/stable.txt`/bin/linux/amd64/kubectl
```

Сделаем бинарный файл исполняемым и переместим утилиту в директорию, по-умолчанию доступную из PATH:

```bash
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

Убедимся что утилита работает корректно:

```bash
kubectl version --client
```

### Установка Minikube

При доступе из сети Интернет следует скачать последнюю версию при помощи команды:

```bash
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
```

Сделаем бинарный файл исполняемым и переместим утилиту в директорию, по-умолчанию доступную из PATH:

```bash
chmod +x ./minikube
sudo mv ./minikube /usr/local/bin/minikube
```

Убедимся что утилита работает корректно:

```bash
minikube version
```

### Запуск Minikube

Minikube запускается при помощи простой команды:

* В рамках настоящей лабораторной работы предполагается использовать драйвер Docker

* 🚨 Указан параметр `--base-image` с указанием образа, доступного из внутренней сети МГТУ им. Н.Э. Баумана

```bash
minikube start --driver=docker
```

В случае, если не требуется использовать внутренние ресурсы МГТУ им. Н.Э. Баумана, параметр `--base-image` можно опустить.

### Проверка работы Minikube

После после подготовки кластера, Minikube автоматически настроет утилиту kubectl путём установки стандартной конфигурации в файле `.kube/config`.

Проверим готовность кластера:

```bash
# Вывод всех доступных нод k8s
kubectl get nodes

# Вывод всех запущенных под во всех пространствах
kubectl get pods --all-namespaces
```

### Особенности работы Minikube

Драйвер Docker Minikube использует подход Docker in Docker (dind), т.е. при запуске Minikube в хост системе создаётся всего один контейнер с именем `minikube`, а все остальные контейнеры и всё управление кластером выполняется в нём.

Подключимся к контейнеру, это можно выполнить двумя способами:

```bash
minikube ssh

docker exec -it minikube bash
```

Внутри этого контейнера доступна утилита `top` для определения всех запущенных процессов, а также утилита `docker`, которая позволит отобразить список запущенных контейнеров и образов.

Следует обратить внимание, что список образов и контейнеров хост системы и minikube контейнера независимы друг от друга.

### Задание 1 | Запуск простого приложения в k8s кластере

В рамках настоящей лабораторной работы сосредоточимся на простейшем приложении:

```bash
docker run -it --rm -p 8080:8080 gcr.io/google-samples/hello-app:1.0
```

Приложение позволяет при помощи web-интерфейса устанавливать то, какой именно контейнер был вызван.

Например, после запуска приложения в Docker и выполнения запроса `curl localhost:8080` будет предоставлен ответ вида:

```text
Hello, world!
Version: 1.0.0
Hostname: 7dc24d1c60bd
```

Где `7dc24d1c60bd` - это идентификатор контейнера.

Данная функциональность пригодится нам при горизонтальном масштабировании нашего сервиса.

Запустим простое приложение в виде пода k8s:

```bash
kubectl run hello --image gcr.io/google-samples/hello-app:1.0
```

**В отчёте следует описать различие результата запуска контейнера в Docker и как пода k8s.**

Отдельно отметим, что запущенный под k8s сетевым образом недоступен ни в хост системе, ни внутри k8s кластера.

Удалить под можно при помощи команды:

```bash
kubectl delete pod hello
```

## Часть 2 | Обеспечение доступности приложения k8s

Созданный нами ранее под был создан в пространстве имён default. Это прстранство имён редко используется на практике поскольку k8s кластер предполагает большой масштаб и совместную работу большого числа специалистов.

Итого, для каждого приложения создаётся своё отдельное пространство имён.

Стоит учитывать, что приложение не обязательно состоит из одного контейнера, оно может быть построено по микросервисной архитектуре.

Начнём запуск приложения с подготовки пространства имён. Имя пространства имён установим соответствующим лоигну пользователя в МГТУ им. Н.Э. Баумана.

Например, `den15u182`.

Kubernetes разрабатывался уже после появления основных концепций DevOps

Kubernetes позволяет использовать не только командный интерфейс, но и применить декларативное описания объектов в форматах yaml или json.

Для задач настоящей лабораторной все описания бдуем добавлять в единый yaml файл `k8s.yaml`.

Для создания пространства имён следует добавить описание:

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: <Название пространства имён>
```

После добавления описания в файл, применим его:

```bash
kubectl apply -f k8s.yaml
```

Убедимся, что пространство имён успешно создано:

```bash
kubectl get namespaces
```

### Задание 2 | Запуск приложения k8s

Запустим приложение, описав его как Deployment, дополним файл `k8s.yaml` следующими строками:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
  namespace: <Название пространства имён>
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: hello
        image: gcr.io/google-samples/hello-app:1.0
        ports:
        - containerPort: 8080
```

Применим изменения:

```bash
kubectl apply -f k8s.yaml
```

Следует обратить внимание, что информационные строки вывода о внесении изменений в приложения содержит описание, что пространство имён не было изменено.

Далее, проверим статусы подготовки приложения:

```bash
# Получить все Deployment
kubectl -n <Название пространства имён> get deployments

# Получить все поды
kubectl -n <Название пространства имён> get pods
```

**В отчёте следует описать сколько подов было создано и их имена.**

### Задание 3 | Подключение приложения к сервису

Развёрнутое выше приложение всё ещё недоступно в сетевом плане.

Для обеспечения доступности приложений следует использовать сущность Service.

Service обеспечивает доступность приложения в режимах (описывается полем `type`):

* `ClusterIP` - тип по-умолчанию, присваивает внутренний IP сервису из пула доступных, используется для того чтобы иметь доступ к сервису внутри кластера, но не извне его.

* `NodePort` - настраивает уникальный порт на каждой из нод кластера для доступа к сервису, используется для простого доступа к приложениям.

* `LoadBalancer` - позволяет присвоить внешний IP адрес сервису, обращание только к нему приведёт к передаче запроса в сервис, используется для захвата стандартных портов.

Следует отметить, что Service вносит изменения на каждой ноде k8s кластера, что позволяет получить доступ к сервису из любой ноды. Однако, следует учитывать временные задержки в случае обращения к ноде, на которой не развёрнуто приложение.

Опишем в файле `k8s.yaml` сервис типа NodePort:

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: hello
  namespace: <Название пространства имён>
spec:
  selector:
    app: hello
  type: NodePort
  ports:
  - port: 8080
    targetPort: 8080
```

После применения дополненного файла проверим корректность созданного сервиса:

```bash
kubectl -n <Название пространства имён> get services
```

Вывод будет иметь вид:

```text
NAME    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
hello   NodePort   10.105.85.253   <none>        8080:32052/TCP   14s
```

Здесь порт `32052` будет являться портом, при обращении к которому будет выполнен запрос к сервису и, следовательно, приложению.

Следует выполнить доступ с использованием ip адреса, присвоеному ноде, т.е. компьютеру, на котором выполняется задание. Этот адрес поможет узнать утилита `ip addr show`.

Однако, Minikube предоставляет автоматизацию для получения подобных адресов:

```bash
minikube service hello -n <Название пространства имён> --url
```

Для доступа к сервису следует выполнить обращение при помощи утилиты `curl` полученному выше адресу.

Следует автоматизировать 10-20 обращений к сервису средствами языка bash и сделать вывод о принципе выбора очередного пода, к которому будет в конечном итоге сделано обращение

**В отчёт следует занести код полученного сценария обращения к сервису и полученые выводы.**

Особенностью применения пространст имён k8s является то, что можно удалить все ресурсы в пространстве имён путём удаления только пространства имён.

```bash
kubectl delete ns <Название пространства имён>
```

## Часть 3 | Развёртывание БД в Kubernetes

Развёртывание БД является примером развёртывания приложения с сохранением состояния (StateFull).

Для сохранения состояния, т.е. данных, применяется Persistent Volumes и Persistent Volume Claims.

Kubernetes API в зависимости от особенностей CSI позволяет автоматизировать создание PV и PVC.

### Задание 4 | Развёртывание БД как StateFull Set

Для развёртывания подготовлен файл `postgres-db.yaml`, следует применить его в k8s.

Исследуем полученные объекты:

```bash
kubectl -n postgres-db get sts

kubectl -n postgres-db get po

kubectl -n postgres-db get pv

kubectl -n postgres-db get pvc

kubectl -n postgres-db get svc
```

**В отчёте следует отметить какие объекты были возвращены при обращении выполнении указанных выше операций.**

Для того чтобы обеспечить работу LoadBalancer, который описан в файле `posgres.yaml` следует обеспечить туннелирование запросов minikube:

```bash
minikube tunnel
```

Далее, при повторном запросе получения сервисов, будет указано External IP адрес, при обращении к которому будет выполнен запрос к PostgreSQL.

### Задание 5 | Идентификация и устранение проблем подключения к StateFull Set

Однако, приведённый метод развёртывания имеет проблемы идентификации к какому именно поду будет выполнено обращение.

Исследуем эти проблемы, для этого следует заменить образ пода Postgres на использованный нами в первых опытах `gcr.io/google-samples/hello-app:1.0`, а также изменить используемые приложением порты.

Далее, следует автоматически выполнить 10-20 запросов к сервису и определить к каким подам будет выполнен запрос.

Для устранения выявленных проблем следует изменить описание сервиса - организовать сервис для каждого из подов:

```yaml
---
# PostgreSQL StatefulSet Service 0
apiVersion: v1
kind: Service
metadata:
  name: postgres-db-lb-0
  namespace: postgres-db
spec:
  selector:
    statefulset.kubernetes.io/pod-name: postgresql-db-0
  type: LoadBalancer
  ports:
  - port: <port>
    targetPort: <port>

---
# PostgreSQL StatefulSet Service 1
apiVersion: v1
kind: Service
metadata:
  name: postgres-db-lb-1
  namespace: postgres-db
spec:
  selector:
    statefulset.kubernetes.io/pod-name: postgresql-db-1
  type: LoadBalancer
  ports:
  - port: <port>
    targetPort: <port>
```

**В отчёте следует указать внесённые изменения в файле `postgres-db.yaml`, а также результаты выполнения запросов к сервису StateFull Set**

ℹ️ В реальных приложениях, которые развёртываются с применением StateFull Set используются дополнения в программном коде, которые позволяют использовать приложение в кластерном режиме.

## Часть 4 | Использование Ingress

Ingress - это описание, которое позволяет при помощи развёртываемого отдельно реверсивного проки можно описать пути до конкретных приложений.

Т.е. используя единую ip адрес и доменное имя при помощи описания путей доступа к конкретным микросервисам объединить их, в стиле, как это выполнялось в ЛБ2 с развртывание Nginx.

Для начала работы с Ingress следует его установить:

```bash
minikube addons enable ingress --images='IngressController=ingress-nginx/controller:v1.11.2,KubeWebhookCertgenCreate=ingress-nginx/kube-webhook-certgen:v1.4.3,KubeWebhookCertgenPatch=ingress-nginx/kube-webhook-certgen:v1.4.3'
```

Флаг `--images` следует опустить если не используются внутренние ресурсы МГТУ им. Н.Э. Баумана.

Далее, аналогично ЛБ2, сгенерируем самоподписанные сертификаты:

```bash
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -sha256 -days 365 -nodes
```

И добавим их как секреты в используемое нами пространство имён:

```bash
kubectl -n <Название пространства имён> create secret tls certs --cert=cert.pem --key=key.pem
```

### Задание 6 | Подключение Ingress

Добавим описание Ingress в файле `k8s.yaml`:

```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello
  namespace: <Название пространства имён>
spec:
  ingressClassName: nginx
  rules:
    - host: hello-world.example
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: hello
                port:
                  number: 8080
  tls:
    - hosts:
      - hello-world.example
      secretName: certs
```

Выполним запрос к сервису через Ingress:

```bash
curl --cacert cert.pem --resolve "hello-world.example:443:$( minikube ip )" -i https://hello-world.example
```

**В отчёте следует указать варианты запросов и ответов в случае запроса по протоколу http (порт 80) и без указания сертификатов, описать получившиеся результаты.**

## Содержание отчёта

1. [`Цель и задачи лабораторной работы`](#цель-и-задачи-лабораторной-работы)
2. [`Задание 1 | Запуск простого приложения в k8s кластере`](#задание-1--запуск-простого-приложения-в-k8s-кластере)
3. [`Задание 2 | Запуск приложения k8s`](#задание-2--запуск-приложения-k8s)
4. [`Задание 3 | Подключение приложения к сервису`](#задание-3--подключение-приложения-к-сервису)
5. [`Задание 4 | Развёртывание БД как StateFull Set`](#задание-4--развёртывание-бд-как-statefull-set)
6. [`Задание 5 | Идентификация и устранение проблем подключения к StateFull Set`](#задание-5--идентификация-и-устранение-проблем-подключения-к-statefull-set)
7. [`Задание 6 | Подключение Ingress`](#задание-6--подключение-ingress)

## Полезные ссылки

### Полное удаление кластера Minikube

Для полного удаления всех ресурсов Minikube, а также закешированных образов, следует:

```bash
minikube delete --all --purge
```

### Helm - средство управления кластером k8s

Очень распространённым является ипользование альтернативной утилиты для работы с Kubernetes API - Helm.

В дальнейшей работе будет полезно ознакомиться с её возможностями: шаблонизацией проекта, а также удобными средствами отката приложений.

[Официальный сайт Helm](https://helm.sh/)
