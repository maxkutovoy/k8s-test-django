# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

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

## Запуск сайта в kubernetes (на примере minikube)

Minikube и kubectl должны быть установлены и запущены.

Для запуска сайта в kubernetes кластере необходимо [задать переменные окружения](#переменные-окружения) в файле `django-cm.yml` в разделе `data`. И загрузить их в кластер командой:

```sh
kubectl apply -f django-cm.yml
```
Внимание! Чтобы данные вашего проекта не попали в репозиторий отключите отслеживание изменений в файле `django-cm.yml` командой:

```sh
git rm --cached django-cm.yml
```

Создать локальный Docker образ проекта командой:

```sh
docker -f ./backend_main_django/Dockerfile -t k8s_test_dajngo .
```

Загрузить созданный образ в кластер. Команда для minikube:

```sh
minikube image load k8s_test_dajngo
```

Запустить deployment файл:

```sh
kubectl apply -f django-deployment.yml
```

Запустить сервис ingress:

```sh
kubectl apply -f ingress.yml
```

Чтобы сайт был доступен по адресу [http://star-burger.test](http://star-burger.test) нужно настроить локальный файл `/etc/hosts`. Для этого можно использовать команду:

```sh
echo "$(minikube ip) star-burger.test" | sudo tee -a /etc/hosts
```
