# Развёртывание dev-версии в кластере Yandex Cloud

Примеры всех манифестов вы можете найти [здесь](../../../kubernetes/dev/yc-sirius/edu-evil-panini).

## Подготовка к развёртыванию

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

1. Проверьте, присутствует ли в вашем `namespace` секрет с ssl сертификатом для PostgreSQL. Если присутствует, переходите сразу к п.4. Если нет, то получите ваш ssl сертификат [по этой инструкции](https://yandex.cloud/ru/docs/managed-postgresql/operations/connect#get-ssl-cert).
2. Создайте YAML конфигурацию для `Secret` с сертификатом:
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: ssl-cert-secret
    data:
      root.crt: <base64_encoded_ssl_certificate>
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
                secretName: ssl-cert-secret  # название сущности Secret с сертификатом
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

## Запуск nginx

1. Создайте YAML конфигурацию для `Deployment`
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-deployment
      labels:
        app.kubernetes.io/name: nginx
        app.kubernetes.io/instance: nginx-dev
        app.kubernetes.io/version: '1.0.0'
        app.kubernetes.io/component: backend
        app.kubernetes.io/part-of: k8s-test-django
        app.kubernetes.io/managed-by: manual
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
            - name: nginx
              image: nginx:1.14.2
              imagePullPolicy: IfNotPresent
              ports:
                - containerPort: 80
    ```

2. Создайте сущность `Deployment`
    ```bash
    kubectl apply -f path/to/nginx/deployment.yaml
    ```

3. Создайте YAML конфигурацию для `Service`
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: nginx-service
      labels:
        app.kubernetes.io/name: nginx-service
        app.kubernetes.io/instance: nginx-service-dev
    spec:
      selector:
        app: nginx
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
          nodePort: 30331  # NodePort, на который ALB распределяет запросы от вашего домена
      type: NodePort
    ```
4. Создайте сущность `Service`
    ```bash
    kubectl apply -f path/to/nginx/service.yaml
    ```
5. Nginx будет доступен по вашему домену. Обычно это домен вида `<your_namespace>.sirius-k8s.dvmn.org`.