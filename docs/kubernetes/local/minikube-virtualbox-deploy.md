# Развёртывание сайта в локальном Minikube кластере

Примеры всех манифестов вы можете найти [здесь](../../../kubernetes/local/minikube-virtualbox/).

## Установка postgreSQL в кластер

1. Установите [Helm](https://helm.sh/) для вашей ОС
2. Установите postgreSQL с помощью [Helm](https://artifacthub.io/packages/helm/bitnami/postgresql):
    ```bash
    helm install <postgres_pod_name> \
    --set auth.username=<your_username> \
    --set auth.password=<your_password> \
    --set auth.database=<your_db_name> \
    oci://registry-1.docker.io/bitnamicharts/postgresql
    ```
3. Проверьте корректность установки с помощью:
    ```bash
    kubectl get pod  # в списке должен быть под с postgreSQL
    kubectl get pvc  # в списке должен быть Volume, привязанный к поду postgreSQL
    kubectl get svc  # в списке должен быть как минимум один сервис, связанный с postgreSQL
    ```
4. Для подключения Django к СУБД используйте в качестве имени хоста имя Service, связанного с подом postgreSQL, для которого указан ClusterIP.
    ```bash
    ❯ kubectl get svc
    NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
    django-app-service          ClusterIP   10.102.94.152   <none>        80/TCP     22h
    kubernetes                  ClusterIP   10.96.0.1       <none>        443/TCP    5d3h
    mk-postgres-postgresql      ClusterIP   10.109.39.88    <none>        5432/TCP   89m  # в URL СУБД указать имя этого сервиса
    mk-postgres-postgresql-hl   ClusterIP   None            <none>        5432/TCP   89m
    ```

    В данном случае `DATABASE_URL` должен быть следующего вида: `postgres://<your_username>:<your_password>@mk-postgres-postgresql:5432/<your_db_name>`

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
      DATABASE_URL: 'base64_database_url'
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
  
## Создание Ingress
1. Пропишите свою конфигурацию в YAML файл:
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: django-app-ingress
      annotations:
        nginx.ingress.kubernetes.io/rewrite-target: /
    spec:
      rules:
        - host: star-burger.test  # имя вашего хоста
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: django-app-service  # ваш ClusterIP сервис для deployment
                    port:
                      number: 80  # порт, на котором работает сервис
    ```
2. Подключите аддон ingress для вашего кластера:
    ```bash
    minikube addons enable ingress
    ```
3. Создайте сущность `Ingress` в вашем кластере
    ```bash
    kubectl apply -f path/to/ingress.yaml
    ```
4. (опционально) Пропишите, если нужно, ваш хост в файле hosts вашей ОС:
    ```hosts
    <minikube_ip> your_host
    ```


## Создание Deployment
1. Загрузите ваш Docker Image внутрь кластера [любым удобным вам способом](https://minikube.sigs.k8s.io/docs/handbook/pushing/)

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
              image: django_app:latest  # ваш Docker образ
              imagePullPolicy: IfNotPresent
              envFrom:
                - secretRef:
                    name: django-secrets  # имя сущности Secrets
                - configMapRef:
                    name: django-config  # имя сущности ConfigMap
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
              image: django_app:latest  # имя вашего образа
              imagePullPolicy: IfNotPresent
              command: ['python', 'manage.py', 'migrate', '--noinput']
              envFrom:
                - secretRef:
                    name: django-secrets  # имя сущности Secrets
                - configMapRef:
                    name: django-config  # имя сущности ConfigMap
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
2. Выполните вход в любой из подов, принадлежащих deployment с джанго:
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
      type: ClusterIP
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
                  image: django_app:latest
                  command: ['python', 'manage.py', 'clearsessions']
                  imagePullPolicy: IfNotPresent
                  envFrom:
                    - secretRef:
                        name: django-secrets
                    - configMapRef:
                        name: django-config
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