# Развёртывание dev-версии в кластере Yandex Cloud

Примеры всех манифестов вы можете найти [здесь](../../../kubernetes/dev/yc-sirius/edu-evil-panini).

## Получение доступа к кластеру

1. Получите доступ к кластеру у его администратора.
2. Установите и инициализируйте [Yandex Cloud CLI](https://yandex.cloud/ru/docs/cli/quickstart#install).
3. Добавьте учётные данные кластера в конфигурационный файл kubectl:
    ```bash
    yc managed-kubernetes cluster get-credentials --id cat528346gdueh53ts39 --external
    ```
4. Измените ваш текущий `namespace` с `default` на тот, который настроил для вас администратор.
    ```bash
    kubectl config set-context --current --namespace=<your_namespace>
    ```

    **или при каждом дальнейшем запуске `kubectl` добавляйте параметр `--namespace=<your_namespace>` или `-n <your_namespace>`.**


## Подключение к PostgreSQL

1. Проверьте, присутствует ли в вашем `namespace` секрет с ssl сертификатом для PostgreSQL (вероятнее всего, он будет называться `pg-root-cert`). Если присутствует, переходите сразу к п.4. Если нет, то получите ваш ssl сертификат [по этой инструкции](https://yandex.cloud/ru/docs/managed-postgresql/operations/connect#get-ssl-cert).
2. Создайте YAML конфигурацию для `Secret` с сертификатом:
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: pg-root-secret
    data:
      root.crt: <base64_ssl_certificate>
    ```
4. Создайте сущность `Secret` с помощью:
    ```bash
    kubectl apply -f path/to/secret.yaml
    ```
5. Создайте YAML конфигурацию для `Deployment` пода с Ubuntu, с помощью которого вы будете подключаться к СУБД.
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: ubuntu-deployment
      labels:
        app.kubernetes.io/name: ubuntu
        app.kubernetes.io/instance: ubuntu-psql-ssl-test
        app.kubernetes.io/version: '1.0.0'
        app.kubernetes.io/component: backend
        app.kubernetes.io/part-of: k8s-test-django
        app.kubernetes.io/managed-by: manual
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: ubuntu
      template:
        metadata:
          labels:
            app: ubuntu
        spec:
          volumes:
            - name: ssl-cert-volume
              secret:
                secretName: pg-root-secret  # название сущности Secret с сертификатом
                defaultMode: 384  # обязательные права rw------- для сертификатов
          containers:
          - image: ubuntu:22.04
            name: ubuntu
            command: ["/bin/sh"]
            args:
              - -c
              - |
                apt-get update &&
                apt-get install -y postgresql-client &&
                sleep 86400  # задержка отключения пода в 24 часа
            imagePullPolicy: IfNotPresent
            volumeMounts:
              - name: ssl-cert-volume
                readOnly: true
                mountPath: "/root/.postgresql"  # директория, куда будет смонтирован том с сертификатом.
    ```
6. Создайте сущность `Deployment` с помощью:
    ```bash
    kubectl apply -f path/to/ubuntu/deployment.yaml
    ```
7. Зайдите в shell контейнера с помощью:
    ```bash
    kubectl exec -itn <your_namespace> <pod_name> -c <container_name> -- sh -c "clear; (bash || ash || sh)"
    ```
8. Подключитесь к СУБД:
    ```bash
    psql "host=<postgres_host> port=<postgres_port> dbname=<db_name> user=<user_name> password=<user_password>"
    ```

## Сборка docker image и загрузка в Docker Hub

1. Соберите docker образ с помощью команды:
```bash
docker build -t <docker_username>/<image_name>:<commit_hash> <path_to_dockerfile>
```
2. Загрузите образ в Docker Hub
```bash
docker push <docker_username>/<image_name>:<commit_hash>
```

Теперь вы можете использовать этот образ в своих манифестах k8s.

## Добавление переменных окружения

1. Создайте `YAML` файл c чувствительными переменными:

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: django-secrets
    type: Opaque
    data:
      SECRET_KEY: 'base64_secret_key'
      ALLOWED_HOSTS: 'base64_allowed_hosts'
    ```

Все значения переменных должны быть закодированы в `BASE64`.

2. Создайте `YAML` файл с обычными переменными:

    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: django-config
    data:
      DEBUG: 'False'
    ```

3. Создайте сущности `Secret` и `ConfigMap` в вашем кластере:
    ```bash
    kubectl apply -f path/to/secrets.yaml
    kubectl apply -f path/to/configmap.yaml
    ```

## Создание Deployment 

2. Пропишите свою конфигурацию в YAML файл:
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: django-app-deployment
    labels:
      app.kubernetes.io/name: django-app
      app.kubernetes.io/instance: django-app-dev
      app.kubernetes.io/version: '1.0.0'
      app.kubernetes.io/component: backend
      app.kubernetes.io/part-of: k8s-test-django
      app.kubernetes.io/managed-by: manual
    spec:
    replicas: 2
    selector:
      matchLabels:
      app: django-app
    template:
      metadata:
      labels:
      app: django-app
      spec:
      containers:
        - name: django-app
          image: lypavel/django_app:latest  # образ из Docker Hub
          imagePullPolicy: IfNotPresent
          envFrom:
            - secretRef:
              name: django-secrets  # имя сущности Secrets
            - configMapRef:
              name: django-config  # имя сущности ConfigMap
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: postgres  # настроенный администратором Secret с данными для подключения к СУБД
                  key: dsn  # переменная с URL для подключения
          ports:
            - containerPort: 80
    ```
3. Создайте сущность `Deployment` в кластере:
    ```bash
    kubectl apply -f path/to/deployment.yaml
    ```

## Применение миграций

1. Пропишите свою конфигурацию в YAML файл:
    ```yaml
    apiVersion: batch/v1
    kind: Job
    metadata:
    name: django-migrate-job
    spec:
    backoffLimit: 4
    activeDeadlineSeconds: 60
    ttlSecondsAfterFinished: 3600
    template:
      spec:
      containers:
        - name: django-migrate
          image: lypavel/django_app:latest  # образ из Docker Hub
          imagePullPolicy: IfNotPresent
          command: ['python', 'manage.py', 'migrate', '--noinput']
          envFrom:
            - secretRef:
              name: django-secrets
            - configMapRef:
              name: django-config
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: postgres  # настроенный администратором Secret с данными для подключения к СУБД
                  key: dsn  # переменная с URL для подключения
      restartPolicy: OnFailure
    ```

2. Создайте сущность Job. После создания она сразу же запустится и применит миграции.
    ```bash
    kubectl apply -f path/to/migrate/job.yaml
    ```
3. Для отслеживания статуса Job используйте:
    ```bash
    kubectl get job
    ```
    Она должна иметь статус `Completed`.

## Создание superuser

1. Получите список подов с помощью:
    ```bash
    kubectl get pods
    ```
2. Выполните вход в любой из подов, принадлежащих `deployment` с джанго:
    ```bash
    kubectl exec -it <django_pod_name> -- /bin/bash
    ```
3. Создайте superuser с помощью:
    ```bash
    python manage.py createsuperuser
    ```
4. Выйдите из пода:
    ```bash
    exit
    ```

## Создание Service

1. Пропишите свою конфигурацию в YAML файл:
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
    name: django-app-service
    labels:
      app.kubernetes.io/name: django-app-service
      app.kubernetes.io/instance: django-app-service-dev
    spec:
    selector:
      app: django-app
    ports:
      - protocol: TCP
        port: 80
        targetPort: 80
        nodePort: 30331  # NodePort, на который ALB распределяет запросы от вашего домена
    type: NodePort
    ```
2. Создайте сущность Service с помощью:
    ```bash
    kubectl apply -f path/to/service.yaml
    ```

## Регулярная очистка сессий с помощью CronJob

1. Пропишите свою конфигурацию в YAML файл:
    ```yaml
    apiVersion: batch/v1
    kind: CronJob
    metadata:
    name: django-clearsessions-cronjob
    spec:
    schedule: '0 0 7,14,21,28 * *'
    startingDeadlineSeconds: 360
    jobTemplate:
      spec:
      ttlSecondsAfterFinished: 3600
      template:
        spec:
        containers:
          - name: clearsessions
          image: lypavel/django_app:latest  # образ из Docker Hub
          command: ['python', 'manage.py', 'clearsessions']
          imagePullPolicy: IfNotPresent
          envFrom:
            - secretRef:
              name: django-secrets
            - configMapRef:
              name: django-config
          env:
            - name: DATABASE_URL
            valueFrom:
              secretKeyRef:
                name: postgres
                key: dsn
        restartPolicy: OnFailure
      backoffLimit: 4
    ```
2. Создайте сущность CronJob с помощью команды:
    ```bash
    kubectl apply -f path/to/cronjob.yaml
    ```
    Эта задача будет производить очистку сессий 7, 14, 21 и 28 числа каждого месяца (примерно раз в неделю).

3. (опционально) Вы можете запустить задачу в любое время с помощью
    ```bash
    kubectl create job --from=cronjob/<cronjob_name> <job_name>
    ```

## Доступ к сайту

После выполнения всех вышеописанных действий сайт будет доступен по вашему домену. Обычно это домен вида `<your_namespace>.sirius-k8s.dvmn.org`.