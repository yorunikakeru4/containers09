# Лабораторная работа №9: CI для PHP-приложения в контейнере (GitHub Actions)

## Студент: Кроитор Александр

## Группа: IA2403

## Преподаватель: M. Croitor

## Дата: 20-04-2026

## Цель работы: Целью работы является знакомство с методами оптимизации образов.

## Задание: Сравнить различные методы оптимизации образов:

- Удаление неиспользуемых зависимостей и временных файлов
- Уменьшение количества слоев
- Минимальный базовый образ
- Перепаковка образа
- Использование всех методов

### ~~Локально вместо Docker использую Podman (аналогичный API, rootless, daemonless)~~

В этот раз для наглядности буду использовать docker, разницы в весе у них немного, но для более качественного анализа и сравнения, а так же более глубоких вариантов оптимизации, сравнение будет именно с docker

Помимо всего прочего, в проекте использую just для быстрых команд и всего такого, в качестве сайта используется submodule https://github.com/Mhowitt/meme_generator

### Just команды:

- submodules -> создаёт и синхронизирует субмодули

```bash
 docker --version
Docker version 27.3.1, build v27.3.1
```

Создаём Dockerfile.raw

```dockerfile
# create from ubuntu image
FROM ubuntu:latest

# update system
RUN apt-get update && apt-get upgrade -y

# install nginx
RUN apt-get install -y nginx

# copy site
COPY site /var/www/html

# expose port 80
EXPOSE 80

# run nginx
CMD ["nginx", "-g", "daemon off;"]

```

Создаём

```bash
docker image build -t mynginx:raw -f Dockerfile.raw .
```

Время создания:

```bash
________________________________________________________
Executed in   26.36 secs      fish           external
   usr time   15.55 millis  824.00 micros   14.72 millis
   sys time   22.60 millis  566.00 micros   22.03 millis
```

Размер контейнера:

```bash
lab9 sudo  sudo docker images mynginx:raw
REPOSITORY   TAG       IMAGE ID       CREATED              SIZE
mynginx      raw       ed382e943edc   About a minute ago   206MB

Много, 206 мб - слишком
```

Создадим Dockerfile.clean, где почистим зависимости

```dockerfile
# create from ubuntu image
FROM ubuntu:latest

# update system
RUN apt-get update && apt-get upgrade -y

# install nginx
RUN apt-get install -y nginx

# remove apt cache
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# copy site
COPY site /var/www/html

# expose port 80
EXPOSE 80

# run nginx
CMD ["nginx", "-g", "daemon off;"]
```

```bash
docker image build -t mynginx:clean -f Dockerfile.clean .
```

время:

```bash
________________________________________________________
Executed in   25.15 secs      fish           external
   usr time   14.92 millis    0.00 millis   14.92 millis
   sys time   22.52 millis    1.72 millis   20.80 millis
```

Несущественно быстрее

Да и размер ненамного меньше, но меньше

```bash
lab9 sudo  sudo docker images mynginx:clean
REPOSITORY   TAG       IMAGE ID       CREATED              SIZE
mynginx      clean     3d5b9f4f05de   About a minute ago   206MB
```

Уменьшим количество слоёв

```dockerfile
# create from ubuntu image
FROM ubuntu:latest

# update system
RUN apt-get update && apt-get upgrade -y && \
    apt-get install -y nginx && \
    apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# copy site
COPY site /var/www/html

# expose port 80
EXPOSE 80

# run nginx
CMD ["nginx", "-g", "daemon off;"]
```

и запускаем

Время чуть лучше

```bash
________________________________________________________
Executed in   23.31 secs      fish           external
   usr time   11.52 millis    0.02 millis   11.50 millis
   sys time   23.84 millis    1.60 millis   22.25 millis
```

И размер неплохо уменьшился

```bash
lab9 sudo  sudo docker images mynginx:few
REPOSITORY   TAG       IMAGE ID       CREATED          SIZE
mynginx      few       c38c1516256b   35 seconds ago   143MB
```

И делаем уже обычный dockerfile, но уже на основе alpine, минимальный дистрибутивный образ

```dockerfile
# create from alpine image
FROM alpine:latest

# update system
RUN apk update && apk upgrade

# install nginx
RUN apk add nginx

# copy site
COPY site /var/www/html

# expose port 80
EXPOSE 80

# run nginx
CMD ["nginx", "-g", "daemon off;"]
```

Время рекордное

```bash
________________________________________________________
Executed in    5.32 secs      fish           external
   usr time    6.86 millis  884.00 micros    5.98 millis
   sys time   13.88 millis  942.00 micros   12.94 millis
```

Размер минимальный

```bash
◄ 12s lab9 sudo  sudo docker images mynginx:latest                                         git:main⎪⎥ 󱄅 15:11
REPOSITORY   TAG       IMAGE ID       CREATED          SIZE
mynginx      latest    cc7e27ce8699   25 seconds ago   13.4MB
```

Создаём Dockerfile.min

```dockerfile
# create from alpine image
FROM alpine:latest

# update system, install nginx and clean
RUN apk update && apk upgrade && \
    apk add nginx && \
    rm -rf /var/cache/apk/*

# copy site
COPY site /var/www/html

# expose port 80
EXPOSE 80

# run nginx
CMD ["nginx", "-g", "daemon off;"]
```

После всех команд

```bash
docker image build -t mynginx:minx -f Dockerfile.min .
docker container create --name mynginx mynginx:minx
docker container export mynginx | docker image import - myngin:min
docker container rm mynginx
docker image list
```

Получаем минимальный образ в

```bash
lab9 sudo  sudo docker image list | grep mynginx
mynginx                                     minx            086428540858   31 seconds ago   10.5MB
```

Рекорд! Победа, празднуем и по домам. Или же нет?

Есть множество способов оптимизирования Dockerfile,
их прям слишком много, но есть один крайне популярный, docker-slim
Он автоматически занимается перепаковкой образа, более эффективно чем я, я неумёха

Попробуем и с ним оптимизировать!

Нам понадобится пакет **docker-slim**

Образ у нас уже создан, остаётся только его ещё сильнее уменьшить

```bash
sudo slim build --http-probe=false mynginx:latest
```

И мы получаем....

```bash
◄ 7s lab9 sudo  sudo docker image list                                                     git:main⎪⎥ 󱄅 15:28
REPOSITORY                                  TAG             IMAGE ID       CREATED          SIZE
mynginx.slim                                latest          18cb8d19dd27   2 seconds ago    8.79MB
mynginx                                     latest          086428540858   7 minutes ago    10.5MB
```

Прекрасный результат, минус 2 мб, это точно спасёт нас и наш гигантский пет проект по todo листу!

QA time

Q: Какой метод оптимизации образов вы считаете наиболее эффективным?
A: slim показал наибольшую эффективность, однако сам пакет docker-slim весит чуть ли не эквивалетно docker, так что закономерен вывод, что самая высокая значимость это вес минимального базового образа, всё же современная память не настолько дорогая, чтобы гоняться между 20 и 10 мб

Q: Почему очистка кэша пакетов в отдельном слое не уменьшает размер образа?
A: Из-за приниципа неизменности слоёв, т.е. следующий слой не изменяет предыдущий, а лишь записывается поверх первого

Q: Что такое перепаковка образа?
A: Перепаковка образа — это слияние нескольких слоёв Docker в один для уменьшения итогового размера. Тот же slim из примера выше делает это автоматически, через export мы делали схожую идею, но руками
