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

## Запуск тестового сервера `nginx` в `yc`

Выполните команду:

```bash
kubectl apply -f .\deploy\yc-sirius\edu-happy-goldberg\nginx-test.yaml
```

## Запуск тестового pod `postgresql` в `yc` для проверки подключения к БД

Создайте манифест `pg-root-cert`:

```bash
apiVersion: v1
kind: Secret
metadata:
  name: pg-root-cert
  namespace: edu-happy-goldberg #имя вашей node
data:
  root.crt: >-
    #содержимое вашего файла root.crt в base64 
type: Opaque
```

Выполните команду:

```bash
kubectl apply -f .\deploy\yc-sirius\edu-happy-goldberg\psql-test.yaml
```

Зайдите н апод командой, где `ubuntu-psql` имя созданного пода(можно посмотреть командой `kubectl get pods`):

```bash
kubectl exec -it ubuntu-psql bash
```

Для проверки подключения к БД выполните команду, где `database_url` - настройки подключения к БД, если `url` содержит спецсимволы, например в пароле,
то надо их заменить в формате `%00`, например [здесь](https://www.urlencoder.org/):

```bash
psql postgres://database_url?sslmode=require
```

## Запуск проекта в `yc`

........

## Запуск проекта в кластере minikube

### Установка minikube и запуск кластера

Установите [`VirtualBox`](https://www.virtualbox.org/), поднимите простой локальный [K8s Cluster](https://www.youtube.com/watch?v=WAIrMmCQ3hE&list=PLg5SS_4L6LYvN1RqaVesof8KAf-02fJSi&index=2)

### Сборка образов

Выполните командy для сборки образов приложения `django` и `postgres`:

```bash
docker-compose build
```

### Запуск образа `postgres`

Создайте файл `.env` из примера и заполните свои значения для подключения к БД:

example.env
```bash
POSTGRES_DB=test_k8s
POSTGRES_USER=test_k8s
POSTGRES_PASSWORD=your_user_password
```

```bash
cp example.env .env
```

В файле `docker-compose.yml` пропишите IP адрес вашего сервера в настройках контейнера `db`:

```bash
...
  db:
    image: postgres:12.0-alpine
    volumes:
      - db_data:/var/lib/postgresql/data
    env_file:
      - ./.env
    ports:
      - your_server_ip_address:5432:5432
...
```

Запуститие образ:

```bash
docker-compose up db
```

#### Запуск БД в кластере при помощи helm

Установите [`helm`](https://helm.sh/). Создайте `helm chart` для `postgresql`:

##### создание и настройка подключения к `helm chart` для `postgresql` с помощью `chart`:

создайте `helm chart`:

```bash
helm install dj-app-db .\postgresql\
```

если вместо `dj-app-db` укажете другое имя, то измените значение `DATABASE_URL` в `dj_secret.yaml`, предварительно закодировав его в `base64`, например [здесь](https://www.base64encode.org/):

```bash
postgres://postgres:password@dj-app-db.default.svc.cluster.local:5432/postgres
```

##### создание и настройка подключения к `helm chart` для `postgresql` пошагово с помощью команд

```bash
helm install my-release oci://registry-1.docker.io/bitnamicharts/postgresql
```

Настройте подключение к `helm chart`, для этого используйте `databaseurl` вида(предварительно закодировав его в `base64`, например [здесь](https://www.base64encode.org/)):

```bash
postgres://postgres:password@postgresql-1728985862.default.svc.cluster.local:5432/postgres
```

где 
- `default` это `namespace` кластера `minikube`
- `postgresql-1728985862` имя созданного `pod`
- `password` это пароль из `secret` файла `postgresql`, который генерируется при создании `chart`, можно получить его закодированном в `base64` командой:

```bash
kubectl get secret --namespace default postgresql-1728985862 -o jsonpath="{.data.postgres-password}"
```

Перед первым запуском приложения необходимо сделать миграции и создать пользователя, для этого зайдите на под приложения:

```bash
kubectl exec -it django-app-.... /bin/bash
```

Выполните команды:

```bash
python manage.py migrate
python manage.py createsuperuser
```

### Загрузка образа приложения в кластер minikube

Выполните загрузку образа `django_app`:

```bash
minikube image load django_app
```

### Запуск проекта в кластере

#### Создание манифеста и запуск `secret`

Скопируйте манифест с примера:

```bash
cp example.dj_secret.yaml dj_secret.yaml
```

Заполните манифест своими данными `base64`, для кодирования строки используйте команду:

```bash
echo -n 'your_string' | base64
```

Запустите `secret`:

```bash
kubectl apply -f ./secret.yaml
```

#### Запустите проект в кластере

Выполните команду для создания `deployment` и `service`:

```bash
kubectl apply -f ./deploy-dj.yaml
```

#### Запуск ingress

Установите `ingress`:

```bash
kubectl apply -f https://projectcontour.io/quickstart/contour.yaml
```

Запустите `ingress`:

```bash
kubectl apply -f ./ingress.yaml
```

Отредактируйте файл `/etc/hosts`(Linux) или `C:\Windows\system32\drivers\etc\hosts`(на Windows), добавьте запись:

```bash
...
192.168.XX.XX	star-burger.test #where 192.168.XX.XX - it`s your IP in local network
...
```

#### Запуск django-clearsession и django-migrate

В папке `./jobs` расположены манифесты для запуска `django-clearsession` и `django-migrate`.
Для выполнения [настройте расписание](https://cloud.google.com/scheduler/docs/configuring/cron-job-schedules) в файлах `clearsession-job.yaml` и `migrate-job.yaml`(пункт `schedule: "* * * * *"). Затем запустите воркеры:

```bash
kubectl apply -f .\jobs\migrate-job.yaml
kubectl apply -f .\jobs\clearsession-job.yaml
```

#### Запуск проекта с помощью `helm`:

Для запуска проекта выполните установку `helm chart`:

```bash
helm install dj-app .\dj-chart\
```

где `dj-app` название приложения
