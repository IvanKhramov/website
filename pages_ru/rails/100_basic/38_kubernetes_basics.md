---
title: Основы Kubernetes
permalink: rails/100_basic/38_kubernetes_basics.html
---

В этой главе мы рассмотрим основные ресурсы Kubernetes для развертывания приложений и для обеспечения доступа к ним изнутри и снаружи кластера.

## Шаблоны, манифесты и ресурсы

Werf при деплое использует YAML-манифесты, описывающие Kubernetes-ресурсы. Эти манифесты получаются из Helm-шаблонов, которые лежат в `.helm/templates` и `.helm/charts`.

_Helm-шаблон_, описывающий Pod (один из Kubernetes-ресурсов), может лежать в `.helm/templates/pod.yaml` и выглядеть так:
{% raw %}
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: standalone-pod
spec:
  containers:
  - name: main
    image: {{ $.Values.image }}  # Helm-шаблонизация, позволяет параметризовать имя образа для контейнера.
    command: ["tail", "-f", "/dev/null"]
```
{% endraw %}

Перед деплоем этот Helm-шаблон с помощью werf преобразуется в _манифест_, который выглядит так:
{% raw %}
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: standalone-pod
spec:
  containers:
  - name: main
    image: alpine  # Если пользователь в качестве имени образа указал "alpine".
    command: ["tail", "-f", "/dev/null"]
```
{% endraw %}

А уже во время деплоя этот манифест становится _ресурсом_ Pod в Kubernetes-кластере. Посмотреть, как этот ресурс выглядит в кластере можно с помощью команды `kubectl get`:
{% raw %}
```yaml
$ kubectl get pod standalone-pod --output yaml
apiVersion: v1
kind: Pod
metadata:
  name: standalone-pod
  namespace: default
spec:
  containers:
  - name: main
    image: alpine
    command: ["tail", "-f", "/dev/null"]
    ...
status:
  phase: Running
  podIP: 172.17.0.7
  startTime: "2021-06-02T13:17:47Z"
```
{% endraw %}

## Запуск приложений

Использование ресурса Pod — простейший способ запустить в Kubernetes один или несколько контейнеров. В манифесте Pod'а описываются контейнеры и их конфигурация. Но на практике сами по себе Pod'ы стараются не деплоить — вместо них деплоят Controller'ы, которые создают и управляют Pod'ами за вас. Один из таких контроллеров — Deployment. Создавая Pod'ы с помощью Deployment, вы сильно упрощаете управление Pod'ами. Вот некоторые возможности, которые предоставляет Deployment, и которых нет у Pod'ов самих по себе:
* При автоматическом или ручном удалении Pod'а Pod будет перезапущен;
* Большая часть конфигурации Pod'а не может быть обновлена. Чтобы обновить конфигурацию Pod нужно пересоздать. Конфигурацию же Pod'а в Deployment'е можно обновлять без пересоздания Deployment'а;
* Обновление конфигурации Pod'ов произойдет без простоя — при обновлении часть Pod'ов со старой конфигурацией остаётся активной до тех пор, пока новые Pod'ы не начнут успешно запускаться;
* Одним Deployment'ом можно запускать нескольких Pod'ов сразу, в том числе на разных Node'ах;
* И многое-многое другое;

Разные контроллеры имеют разные возможности, связанные с созданием и управлением Pod'ами. Основные контроллеры и их практическое применение:
* [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) — стандарт для деплоя stateless-приложений;
* [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) — стандарт для деплоя stateful-приложений;
* [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) — для деплоя приложений, которые должны быть запущены только по одному экземпляру на каждой Node (агенты для логирования, мониторинга);
* [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/) — для запуска разовых задач в Pod'ах (например, миграции базы данных);
* [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) — для многоразового запуска задач в Pod'ах по расписанию (например, регулярная подчистка чего-либо);

Получить список ресурсов определенного типа в кластере можно с помощью `kubectl get`:
{% raw %}
```bash
kubectl get --all-namespaces pod
kubectl get --all-namespaces deployment
kubectl get --all-namespaces statefulset
kubectl get --all-namespaces job
kubectl get --all-namespaces cronjob
```
{% endraw %}

Также можно получить полную конфигурацию ресурса, если добавить в команду `kubectl get` опцию `--output yaml`:
{% raw %}
```yaml
$ kubectl get --namespace default deployment somedeployment --output yaml
...
kind: Deployment
metadata:
  name: somedeployment
...

$ kubectl get -n default pod somepod -o yaml
...
kind: Pod
metadata:
  name: somepod
...
```
{% endraw %}

Чаще всего вам придется сталкиваться с ресурсом Deployment, поэтому рассмотрим его подробнее. Про остальные типы контроллеров вы можете прочесть в [документации к Kubernetes](https://kubernetes.io/docs/concepts/workloads/).

## Deployment

Создадим в любой директории файл `deployment.yaml`, и опишем в нём Deployment для stateless-приложения:
{% raw %}
```yaml
apiVersion: apps/v1
kind: Deployment  # Тип ресурса.
metadata:
  name: kubernetes-basics-app  # Имя Deployment'а.
spec:
  replicas: 2  # Можно развернуть несколько Pod'ов сразу.
  selector:
    matchLabels:
      app: kubernetes-basics-app
  template:
    metadata:
      labels:
        app: kubernetes-basics-app
    spec:
      terminationGracePeriodSeconds: 60  # Сколько секунд есть у процессов Pod'а на graceful-завершение после получения TERM-сигнала при остановке Pod'а.
      #####################################################################################################
      # Описание конфигурации контейнеров Pod'а:
      #####################################################################################################
      containers:
      - name: main  # Имя первого контейнера.
        image: alpine  # Имя и тег образа контейнера.
        command:
          # Основная команда контейнера, начнёт выполняться при его запуске:
          - sh
          - -ec
          - |
            while true; do
              echo -e "HTTP/1.1 200 OK\n\nAlive.\nOur \$MY_ENV_VAR value is \"$MY_ENV_VAR\"." | nc -l -s 0.0.0.0 -p 80
            done
        ports:
        - containerPort: 80  # Открытый порт, на котором будет слушать приложение.
        env:
        - name: "MY_ENV_VAR"  # Имя дополнительной переменной окружения.
          value: "myEnvVarValue"  # Значение дополнительной переменной окружения.
        resources:
          requests:
            cpu: 50m  # Не запускаться на Node'ах, которые не могут выделить 0.05 CPU.
            memory: 50Mi  # Не запускаться на Node'ах, которые не могут выделить 50МБ RAM.
          limits:
            cpu: 50m  # Hard-лимит на CPU, не позволит контейнеру расходовать больше.
            memory: 50Mi  # Hard-лимит на RAM, при превышении ООМ-киллер остановит процессы контейнера.
        lifecycle:
          preStop:
            exec:
              command: ["/bin/trigger-graceful-shutdown-for-my-app"]  # Запустится перед завершением контейнера.
        startupProbe:  # Проверка готовности контейнера. При нескольких неудачах контейнер перезапустится.
          httpGet:
            path: /startup  # Будет выполняться GET-запрос на http://<PodIP>:3000/startup.
            port: 80
        readinessProbe:  # Проверка готовности контейнера, запускается после startupProbe. При нескольких неудачах трафик перестаёт идти на Pod через Service.
          httpGet:
            path: /readiness
            port: 80
        livenessProbe:  # Проверка работоспособности контейнера, запускается после startupProbe. При нескольких неудачах контейнер перезапускается.
          httpGet:
            path: /liveness
            port: 80
      #####################################################################################################
      # InitContainers используются для разовых задач, выполняющихся перед запуском основных контейнеров.
      # Основные контейнеры Pod'а не запустятся, пока не выполнятся initContainers.
      #####################################################################################################
      initContainers:
        - name: wait-postgres
          image: postgres
          command: ["echo", "pg_isready", "-h", "postgres.example.com"]
```
{% endraw %}
> Этот пример не является примером того, как должен выглядеть production-ready Deployment. Лучшие практики по организации ресурсов для ваших приложений мы рассмотрим в следующих главах. Более подробное описание Deployment доступно в [официальной документации](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

Создадим Deployment-ресурс в кластере на основе созданного файла `deployment.yaml` с помощью `kubectl apply`:
```bash
kubectl apply -f deployment.yaml
```
> В этой главе для деплоя вместо `werf converge` мы будем использовать `kubectl apply`. Он удобен для быстрого и простого создания ресурсов в кластере, но для деплоя реального приложения необходимо использовать `werf converge`.

Убедимся, что наш Deployment создался:
```bash
kubectl get deployment kubernetes-basics-app
```

А теперь понаблюдаем за развертыванием Pod'ов, создаваемых нашим Deployment'ом:
```bash
kubectl get pod -l app=kubernetes-basics-app
```

## Service и Ingress

С помощью Deployment мы можем развернуть наше stateless-приложение, но если пользователям или другим приложениям потребуется связываться с этим приложением изнутри или снаружи кластера, то нам потребуются два дополнительных ресурса — Ingress и Service.

Создадим файл `ingress.yaml`:
{% raw %}
```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
  name: kubernetes-basics-app
spec:
  rules:
  - host: kubernetes-basics-app.example.com  # Домен, запросы на который будут обрабатываться в paths ниже.
    http:
      paths:
      - path: /  # Запросы с префиксом / (все запросы) перенаправятся на порт 80 нашего Service'а.
        backend:
          serviceName: kubernetes-basics-app
          servicePort: 80
```
{% endraw %}

И создадим файл `service.yaml`:
{% raw %}
```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubernetes-basics-app
spec:
  selector:
    app: kubernetes-basics-app  # Этот Service перенаправляет трафик на Pod'ы с этим лейблом.
  ports:
  - name: http
    port: 80  # Перенаправить трафик с 80 порта Service'а на 80 порт Pod'а.
```
{% endraw %}

А теперь создадим ресурсы в кластере на основе этих манифестов:
```bash
kubectl apply -f ingress.yaml -f service.yaml
```

Убедимся, что наши ресурсы создались:
```bash
kubectl get service kubernetes-basics-app
kubectl get ingress kubernetes-basics-app
```

Если несколько упрощать, то эти два ресурса позволят HTTP-пакетам, приходящим на [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/how-it-works/), у которых есть заголовок `Host: kubernetes-basics-app.example.com`, быть перенаправленными на 80-й порт Service'а `kubernetes-basics-app`, а оттуда на 80-й порт одного из Pod'ов нашего Deployment'а. В конфигурации по умолчанию Service будет перенаправлять запросы на все Pod'ы Deployment'а поровну.

Попробуем достучаться до нашего приложения через Ingress:
{% raw %}
```bash
$ curl http://kubernetes-basics-app.example.com
Alive.
Our $MY_ENV_VAR value is "myEnvVarValue".
```
{% endraw %}

При этом Service-ресурсы нужны не только для связи Ingress'ов и приложения. Service-ресурсы также дают возможность ресурсам внутри кластера общаться между собой. При создании Service'а создается доменное имя `<ServiceName>.<NamespaceName>.svc.cluster.local`, доступное изнутри кластера. Также Service доступен и по более коротким доменным именам: `<ServiceName>` при обращении из того же Namespace или `<ServiceName>.<NamespaceName>` при обращении из другого.

Попробуем создать новый контейнер, не имеющий отношения к нашему приложению, и обратиться из него к нашему приложению через Service:
{% raw %}
```bash
$ kubectl run another-kubernetes-basics-app --image=alpine --rm -it -- sh  # Запустим новый контейнер.
/ apk add curl  # Установим curl внутри контейнера.
/ curl http://kubernetes-basics-app  # Обратимся к одному из Pod'ов нашего приложения через Service.
Alive.
Our $MY_ENV_VAR value is "myEnvVarValue".
/ exit
```
{% endraw %}

> Использование Ingress-ресурсов — не единственный способ получить доступ к приложению снаружи кластера. Service'ы типа LoadBalancer и NodePort позволяют предоставить доступ к приложению снаружи и без Ingress'ов. Почитать подробнее про Service'ы вы можете в [официальной документации](https://kubernetes.io/docs/concepts/services-networking/service/). А больше информации про Ingress'ы можно найти [здесь](https://kubernetes.io/docs/concepts/services-networking/ingress/).

## Подчистка

Удалим созданные нами ресурсы, так как они нам больше не понадобятся:
```bash
kubectl delete -f deployment.yaml -f service.yaml -f ingress.yaml
```