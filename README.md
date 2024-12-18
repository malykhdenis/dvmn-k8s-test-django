# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции будут его активно использовать.

## Как запустить сайт для локальной разработки

Запустите базу данных и сайт:

```shell
$ docker compose up
```

В новом терминале, не выключая сайт, запустите несколько команд:

```shell
$ docker compose run --rm web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker compose run --rm web ./manage.py createsuperuser  # создаём в БД учётку суперпользователя
```

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по адресу [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).

## Как вести разработку

Все файлы с кодом django смонтированы внутрь докер-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не требовал постоянно пересборки докер-образа -- достаточно перезапустить сервисы Docker Compose.

### Как обновить приложение из основного репозитория

Чтобы обновить приложение до последней версии подтяните код из центрального окружения и пересоберите докер-образы:

``` shell
$ git pull
$ docker compose build
```

После обновлении кода из репозитория стоит также обновить и схему БД. Вместе с коммитом могли прилететь новые миграции схемы БД, и без них код не запустится.

Чтобы не гадать заведётся код или нет — запускайте при каждом обновлении команду `migrate`. Если найдутся свежие миграции, то команда их применит:

```shell
$ docker compose run --rm web ./manage.py migrate
…
Running migrations:
  No migrations to apply.
```

### Как добавить библиотеку в зависимости

В качестве менеджера пакетов для образа с Django используется pip с файлом requirements.txt. Для установки новой библиотеки достаточно прописать её в файл requirements.txt и запустить сборку докер-образа:

```sh
$ docker compose build web
```

Аналогичным образом можно удалять библиотеки из зависимостей.

<a name="env-variables"></a>
## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Установка базы через Helm

Установка Helm
```
brew install helm
```

Cоздание пода с базой данных
```
helm install my-release oci://registry-1.docker.io/bitnamicharts/postgresql
```

Вход в под
```
kubectl exec -it my-release-postgresql-0 -- /opt/bitnami/scripts/postgresql/entrypoint.sh /bin/bash
```

Создать БД
```
psql
create database <db_name>;
create user <username> with encrypted password <password>;
grant all privileges on database <db_name> to <username>;
```

## Создание Secret в кластере

В директории `kubernetes/` создать файл `dvmn_django_app_secrets.yml`, и заполнить его по примеру `dvmn_django_app_secrets_example.yml`. Каждое значение ключа должно быть закодировано в `base64`.

Перейдите в директорию `kubernetes/`:
```shell
cd kubernetes
```

Создайте объект `Secret` в кластере командой:
```shell
kubectl apply -f dvmn_django_app_secrets.yml
```

## Развертывание в minikube

Запуск deployment для создания контейнеров с приложением
```
kubectl apply -f /kubernetes/dvmn_django_app.yml
```

Запуск ingress для открытия портов
```
kubectl apply -f /kubernetes/dvmn-ingress.yml
```

Запуск job для выполнения миграций
```
kubectl apply -f /kubernetes/dvmn-django-migrate.yml
```

Запуск cronjob для регулярной очистки сессий
```
kubectl apply -f /kubernetes/dvmn-django-clearsessions-cronjob.yml
```

Открыть проект в браузере
```
minikube service django-app
```

## Ссылка на список выделенных ресурсов облачной инфраструктуры

https://sirius-env-registry.website.yandexcloud.net/edu-silly-lamarr.html

## Как задеплоить код

Деплой nginx после подключнеия к кластеру на Yandex Cloud:
```
kubectl apply -f yc-sirius/edu-silly-lamarr/nginx.yaml
```

## Как подготовить dev окружение

Получить SSL ключ ([Инструкция Yandex Cloud](https://yandex.cloud/ru/docs/managed-postgresql/operations/connect)) и закодировать в base64.

Далее полученный результат запишите в секрет
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pg-root-cert
data:
  root.crt: <SSL в base64>
```

Секрет в pode или deploy файле:
```yaml
...
spec:
  volumes:
    - name: secret-volume
      secret:
        secretName: pg-root-cert
  containers:
    - image: ...
      volumeMounts:
        - name: secret-volume
          readOnly: true
          mountPath: "/root/.postgresql"
```
В папке /root/.postgresql файл с SSL сертификатами (название файла будет `root.crt`).
3. Запуск и подключение к POD. 
```shell
kubectl apply -f <манифест poda>
kubectl exec -it <имя poda> -- /bin/bash
```
4. Подключитесь к базе данных.
```shell
psql "host=rc1b-qitwkw0k********.rw.mdb.yandexcloud.net \
  port=6432 \
  sslmode=verify-full \
  dbname=<имя_БД> \
  user=<имя_пользователя> \
  target_session_attrs=read-write"
```

## Сборка и публикация докер-образов

Cборка образа
```
docker build --file Dockerfile --tag <docker_hub_username>/django_app:<tag> ./backend_main_django
```

Публикация образа
```
docker push <docker_hub_username>/django_app:<tag>
```

## Деплой на prod

1. Создать Secret с SSL сертификатами для подключения к PostgreSQL в Yandex Cloud
```
kubectl apply -f yc-sirius/edu-silly-lamarr/ssl_secret.yaml
```

2. Создать Secret для Django
```
kubectl apply -f yc-sirius/edu-silly-lamarr/django_secret.yaml
```

3. Создать Deployment
```
kubectl apply -f yc-sirius/edu-silly-lamarr/deployment.yaml
```

4. Запустить Service для доступа к приложению по адресу домена https://edu-silly-lamarr.sirius-k8s.dvmn.org
```
kubectl apply -f yc-sirius/edu-silly-lamarr/service.yaml
```


5. Запустить Pod для выполнения миграций
```
kubectl apply -f yc-sirius/edu-silly-lamarr/migrate.yaml
```

6. Зайти в запущенный Pod и создать суперюзера для входа в admin
```
kubectl exec -it <pod's name> -- bash
./manage.py createsuperuser
```
