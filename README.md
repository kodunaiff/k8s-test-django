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

### Работа с упрощенным Kubernetes-кластером (Minikube)

Установите [VirtualBox](https://www.virtualbox.org/wiki/Downloads), [Minikube](https://kubernetes.io/ru/docs/tasks/tools/install-minikube/), [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/).
Для запуска Minikube выполните:

```
minikube start --driver=virtualbox --no-vtx-check
```
Чтобы убедиться, что все запустилось выполните команду:
``` kubectl cluster-info ```
Чтобы управлять докером внутри кластера выполните команду в зависимости от вашей ОС:

``` 
minikube docker-env --shell powershell | Invoke-Expression 
```

``` 
eval $(minikube docker-env) 
```

Теперь нам необходимо загрузить докер-образ внутрь нашего кластера.
Пройдите в директорию с вашим DockerFile или файлом docker-compose (я использовал вариант с докер-компос)
Лучше сперва проверить образы в Докере ``` docker images ```.

``` 
docker compose build 
```
Проверьте снова, образ должен появиться не локально , а именно в докере кластера ``` minikube image ls ```.
Результат: 
``` 
docker.io/library/django_app:latest 
```

### Создание Secrets

Перед создаем "Secret", создайте локально файл .env и заполните его. Пример:

```
SECRET_KEY=test
DATABASE_URL=postgres://test...
DEBUG=True
ALLOWED_HOSTS=*
```

Далее выпоните команду создания "Secret" из этой директории, где лежит файл.
``` 
kubectl create secret generic django-secret --from-env-file=.env 
```

Дополните свой манифест файл:
```
envFrom:
        - secretRef:
            name: django-secret
```
Для просмотра списка секретов, выполните:  
``` kubectl get secrets ```

### Работа с Ingress

Minikube поставляется с предустановленным Ingress-контроллером, но он по умолчанию отключен. Чтобы его включить, выполните команду:
```
minikube addons enable ingress
```
Примените Ingress и Deployment:
```
kubectl apply -f ingress_v1.yaml
kubectl apply -f deploy_django_v2.yaml
```

Добавьте запись в /etc/hosts , если используете Windows файл лежит в этой директории (C:\Windows\System32\drivers\etc). Добавьте в файл такое содержание:
```
<minikube-ip> example.com
```

### Работа с CronJob

Для очистки сессий раз в месяц, воспользуйтесь командой:
```
kubectl apply -f kubectl apply -f cron_dj_clearsession.yaml

kubectl get cronjobs
```


Для запуска в любой момент времени Job из CronJob выполните команду:

```
kubectl create job --from=cronjob/django-clearsessions dj-clear -n default

kubectl get jobs
```


### Работа с Job для выполнения команды migrate

Для запуска management-команд Django, таких как ./manage.py migrate, 
рекомендуется использовать объекты Kubernetes такие как Job для одноразовых задач 
или CronJob для периодических задач.

Для запуска команды migrate используйте манифест для Job:

```
kubectl apply -f migrate_django.yaml
```

Для проверки логов, нужно узнать имя пода. Выполните следующие команды:

```
kubectl describe job django-migrate-job
```

В разделе Events находим имя пода, а дальше:
 
```
kubectl logs django-migrate-job-t7n4b (замените на ваше имя пода)
```


### Работа с Базой Данных используя Helm

Установите [Helm](https://github.com/helm/helm/releases)

Добавьте репозиторий чартов Bitnami в Helm, выполнив:
```
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Обновите информацию о доступных чартах, выполнив:
```
helm repo update
```

Установите PostgreSQL с помощью Helm.
Вы можете установить PostgreSQL, используя команду helm install.

```
helm install my-postgres oci://registry-1.docker.io/bitnamicharts/postgresql --set auth.username=kodu --set auth.password=****** --set auth.database=kodudb
вместо * установите пароль
```

что бы проверить и зайти в бд  под пользователем используйте команды:
```
kubectl exec -it my-postgres-postgresql-0 -- /bin/bash
psql -U kodu -d kodudb
далее пароль..
```

Что бы подключиться к БД измените файл .env:
```
DATABASE_URL=postgres://kodu:[your-password]@my-postgres-postgresql:5432/kodudb
```

Далее обновите Secrets или создайте новый, но тогда не забудьте исправить в манифестах секцию с envFrom

```
kubectl create secret generic new-django-secret --from-env-file=.env
```

Теперь обновите Deploy и запустите миграции:

```
kubectl delete -f deploy_django_v2.yaml
kubectl apply -f deploy_django_v2.yaml
kubectl apply -f migrate_django.yaml
```

Зайдите на сайт, все должно работать.