# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию

Запустите базу данных и сайт:

```shell-session
$ docker-compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell-session
$ docker-compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker-compose run web ./manage.py createsuperuser
```

Для тонкой настройки Docker Compose используйте переменные окружения. Их названия отличаются от тех, что задаёт docker-образа, сделано это чтобы избежать конфликта имён. Внутри docker-compose.yaml настраиваются сразу несколько образов, у каждого свои переменные окружения, и поэтому их названия могут случайно пересечься. Чтобы не было конфликтов к названиям переменных окружения добавлены префиксы по названию сервиса. Список доступных переменных можно найти внутри файла [`docker-compose.yml`](./docker-compose.yml).

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).


## Запуск в Kebernetes

Запустите где-нибудь снаружи Postgres. Для теста можно запустить в докере командой:
```bash
docker compose -f docker-compose.yml -f docker-compose.onlydb.yml up -d
```

Переименуйте файл `kubernetes/django-app-env.yaml.example` в `kubernetes/django-app-env.yaml`:
```bash
cp django-app-env.yaml.example django-app-env.yaml
```

 Установите значения [переменных окружения](#переменные-окружения) в файле `kubernetes/django-app-env.yaml`.

 Последовательно выполните следующие команды:
 ```bash
 kubectl apply -f kubernetes/django-app-env.yaml
 kubectl apply -f kubernetes/django-app-service.yaml
 ```

 TCP-порт, на котором работает сервис, можно посмотреть в выводе команды:
 ```
 kubectl get services django-app-service
 ```
 или, если используете minikube:
 ```bash
 minikube service django-app-service --url
 ```

Статус запуска подов можно посмотреть командой:
```bash
kubectl get pods
```

Если Вы изменили переменные окружения, необходимо перезапустить Deployment командой:
```bash
kubectl rollout restart deployment django-app-deployment
```

Для применения миграций, запустите `Job` командой:
```bash
kubectl apply -f kubernetes/django-app-migrate.yaml
```

### Запуск Ingress

1. В случае, если используется в качестве кластера `minikube`, в файле `/ets/hosts` укажите ip-адрес `minikube` и доменное имя `star-burger.test`
```bash
echo "$(minikube ip) star-burger.test" | sudo tee -a /etc/hosts
```

2. Запустите Ingress командой:
```bash
kubectl apply -f kubernetes/django-app-ingress.yaml
```

Сайт будет доступен по адресу `star-burger.test`

Можно указать произвольное доменное имя, для этого нужно его так же указать в файле `kubernetes/django-app-ingress.yaml` и в переменной окружения `ALLOWED_HOSTS` в файле `kubernetes/django-app-env.yaml`.

### Удаление устаревших сессий

Для удаления устаревших сессий, запустите `CronJob` командой:
```bash
kubectl apply -f kubernetes/django-app-cronjobs.yaml
```

По умолчанию, удаление устаревших сессий выполняется раз в неделю.
