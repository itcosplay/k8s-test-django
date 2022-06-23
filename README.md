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



## Деплой в локальном кластере Kubernetes Minikube
Предварительно должны быть установлены:
- [kubectl](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/)
- [minikube](https://minikube.sigs.k8s.io/docs/start/)
- [VirtualBox](https://www.virtualbox.org/)
- [Helm](https://helm.sh/docs/intro/install/)

### Деплой по шагам:  
1. Кластер kubernetes  
Cоздаем кластер
```
minikube start --driver=virtualbox
```

2. База данных и Helm  
Добавляем необходимый чарт для Postgresql
```
helm repo add bitnami https://charts.bitnami.com/bitnami
```
```
helm install psql-db bitnami/postgresql
```
Создайте БД с именем, именем пользователя и паролем следуя инструкциям.  

3. Загружаем образ Django в кластер.  
```
minikube image load <docker-image-name:tag>
```

4. Environments  
В папке Kubernetes создаем файл django-config.yaml следующего вида:  
```yaml
apiVersion : v1
kind : ConfigMap
metadata :
  name: django-config
  labels:
    env: production
data :
  SECRET_KEY : 'YOUR-SECRET-KEY'
  DATABASE_URL : 'postgres://USER:PASSWORD@<name-from-step-2>-postgresql:5432/POSTGRES_DB_NAME'
  ALLOWED_HOSTS : 'IP вашего minikube'
```
Создаем `СonfigMap` командой:
```
kubectl apply -f ./kubernetes/django-config.yaml
```
5. Ingress  
Включаем надстройку minikube ingress:
```
minikube addons enable ingress
```
Создаем объект `Ingress`:
```
kubectl apply -f ./kubernetes/ingress-host.yaml
```
Добавляем в конец файла [hosts](https://2domains.ru/support/hosting/hosts-file) строчку `<minikube ip> star-burger.test`  

Проверяем сайт по адресу [http://star-burger.test](http://star-burger.test)

6. Managment команды  
Запускаем миграции БД:
```
kubectl apply -f kubernetes/migrate.yaml
```
Удаляем job:
```
kubectl delete job django-migrate
```

7. Сессии django  
Запускаем регулярное удаление сессий раз в месяц:
```
kubectl apply -f kubernetes/clearsessions.yaml
```


Код написан в учебных целях — это урок в курсе по Python и веб-разработке на сайте [Devman](https://dvmn.org).