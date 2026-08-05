# Nginx на Windows через Docker — команды установки и создания окружения

Этот документ собирает команды, которые использовались в ходе настройки учебного проекта:

- запуск Nginx в Docker;
- переход на Docker Compose;
- подключение HTML, конфигурации и логов;
- создание собственного Docker-образа;
- сборка модуля GeoIP2;
- подключение базы MaxMind GeoLite2;
- настройка HTTPS и HTTP/2;
- диагностика проблем.

> Команды рассчитаны преимущественно на **Git Bash в Windows**.  
> В Git Bash перенос строки выполняется символом `\`, а не обратной кавычкой PowerShell `` ` ``.

---

## 1. Проверка Docker

### Проверить установленную версию Docker

```bash
docker --version
```

Показывает установленную версию Docker CLI.

### Проверить Docker Compose

```bash
docker compose version
```

Проверяет, доступна ли современная команда Docker Compose.

### Проверить состояние Docker Engine

```bash
docker info
```

Показывает информацию о Docker Engine, контейнерах, образах, сетях и хранилище.

Если команда возвращает ошибку подключения, обычно Docker Desktop не запущен.

---

## 2. Первый запуск Nginx

### Скачать образ и создать контейнер

```bash
docker run --name nginx-course -p 8000:80 -d nginx:stable
```

Разбор:

- `docker run` — создаёт и запускает новый контейнер;
- `--name nginx-course` — задаёт имя контейнера;
- `-p 8000:80` — связывает порт `8000` Windows с портом `80` контейнера;
- `-d` — запускает контейнер в фоне;
- `nginx:stable` — официальный stable-образ Nginx.

После запуска сайт доступен по адресу:

```text
http://localhost:8000
```

### Посмотреть работающие контейнеры

```bash
docker ps
```

Показывает только запущенные контейнеры.

### Посмотреть все контейнеры

```bash
docker ps -a
```

Показывает и работающие, и остановленные контейнеры.

### Посмотреть логи контейнера

```bash
docker logs nginx-course
```

Показывает стандартный вывод и ошибки контейнера.

### Следить за логами в реальном времени

```bash
docker logs -f nginx-course
```

Флаг `-f` означает `follow`.

Остановка просмотра:

```text
Ctrl + C
```

### Остановить контейнер

```bash
docker stop nginx-course
```

Контейнер остаётся созданным, но Nginx прекращает работу.

### Запустить остановленный контейнер

```bash
docker start nginx-course
```

Запускает ранее созданный контейнер.

### Удалить контейнер

```bash
docker rm -f nginx-course
```

- `rm` — удалить контейнер;
- `-f` — сначала принудительно остановить его, если он работает.

Образ Nginx при этом не удаляется.

---

## 3. Работа внутри контейнера

### Открыть shell контейнера

```bash
docker exec -it nginx-course sh
```

Разбор:

- `docker exec` — выполняет команду внутри работающего контейнера;
- `-i` — оставляет стандартный ввод открытым;
- `-t` — создаёт интерактивный терминал;
- `sh` — запускает оболочку внутри контейнера.

В официальном минимальном образе Nginx обычно есть `sh`, но нет `bash`.

### Выйти из контейнера

```bash
exit
```

### Посмотреть файлы Nginx

Внутри контейнера:

```sh
ls -la /usr/share/nginx/html
```

Стандартная директория статических файлов Nginx.

```sh
cat /etc/nginx/nginx.conf
```

Показывает основной конфигурационный файл.

```sh
cat /etc/nginx/conf.d/default.conf
```

Показывает конфигурацию виртуального сервера.

---

## 4. Создание структуры проекта

### Создать рабочую папку

```bash
mkdir -p /d/backend/nginx-course-udemy
cd /d/backend/nginx-course-udemy
```

`mkdir -p` создаёт директорию, включая недостающие родительские директории.

### Создать папки проекта

```bash
mkdir -p html nginx logs geoip certs
```

Назначение:

```text
html/   — HTML, CSS, JavaScript и изображения
nginx/  — Dockerfile и конфигурация Nginx
logs/   — локальные access/error логи
geoip/  — база GeoLite2
certs/  — TLS-сертификат и приватный ключ
```

### Создать файлы

```bash
touch html/index.html
touch nginx/default.conf
touch nginx/nginx.conf
touch nginx/Dockerfile
touch compose.yml
```

`touch` создаёт пустой файл, если его ещё нет.

---

## 5. Подключение HTML через bind mount

### Запуск с локальной HTML-папкой

```bash
docker run \
  --name nginx-course \
  -p 8080:80 \
  -v "${PWD}/html:/usr/share/nginx/html:ro" \
  -d nginx:stable
```

Разбор mount:

```text
${PWD}/html                 — папка на Windows
/usr/share/nginx/html       — папка внутри контейнера
:ro                         — контейнер может только читать
```

Изменения в локальном `html/index.html` сразу видны контейнеру.

### Важное различие оболочек

Git Bash:

```bash
docker run \
  --name example \
  ...
```

PowerShell:

```powershell
docker run `
  --name example `
  ...
```

В Git Bash обратная кавычка PowerShell приводит к ошибкам вида:

```text
bash: --name: command not found
```

---

## 6. Переход на Docker Compose

### Минимальный `compose.yml`

```yaml
services:
  nginx:
    image: nginx:stable
    container_name: nginx-course
    restart: unless-stopped

    ports:
      - "8080:80"

    volumes:
      - ./html:/usr/share/nginx/html:ro
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      - ./logs:/var/log/nginx
```

### Проверить YAML и итоговую конфигурацию

```bash
docker compose config
```

Полезно выполнять перед запуском. Команда обнаруживает ошибки отступов и YAML-синтаксиса.

### Запустить сервисы

```bash
docker compose up -d
```

Создаёт сеть и контейнеры из `compose.yml`.

### Запустить с пересозданием контейнера

```bash
docker compose up -d --force-recreate
```

Нужно, когда изменились:

- ports;
- volumes;
- environment;
- другие параметры контейнера.

Обычный `restart` такие изменения не применяет.

### Посмотреть состояние сервисов

```bash
docker compose ps
```

### Посмотреть логи Nginx

```bash
docker compose logs nginx
```

### Следить за логами

```bash
docker compose logs -f nginx
```

### Остановить и удалить сервисы

```bash
docker compose down
```

Удаляет контейнеры и сеть проекта.

### Удалить также Compose-volumes и лишние контейнеры

```bash
docker compose down --volumes --remove-orphans
```

Не удаляет локальные bind mount-папки проекта.

---

## 7. Проверка и применение конфигурации Nginx

### Проверить конфигурацию работающего контейнера

```bash
docker compose exec nginx nginx -t
```

Ожидаемый результат:

```text
syntax is ok
test is successful
```

### Проверить конфигурацию через временный контейнер

```bash
docker compose run --rm nginx nginx -t
```

Полезно, когда основной контейнер не запускается или находится в restart-loop.

- `run` — создаёт временный контейнер;
- `--rm` — удаляет его после выполнения.

### Вывести всю активную конфигурацию

```bash
docker compose exec nginx nginx -T
```

`-T` проверяет конфигурацию и печатает все подключённые файлы.

### Перезагрузить Nginx без остановки контейнера

```bash
docker compose exec nginx nginx -s reload
```

Правильная последовательность:

```bash
docker compose exec nginx nginx -t &&
docker compose exec nginx nginx -s reload
```

Вторая команда запускается только при успешной проверке первой.

---

## 8. Сохранение логов на Windows

### Подключить локальную папку логов

В `compose.yml`:

```yaml
volumes:
  - ./logs:/var/log/nginx
```

У этого mount нельзя ставить `:ro`, потому что Nginx должен записывать файлы.

### Настроить логи в `default.conf`

```nginx
access_log /var/log/nginx/course-access.log main;
error_log /var/log/nginx/course-error.log warn;
```

### Посмотреть локальные логи

```bash
ls -la logs
```

```bash
cat logs/course-access.log
```

```bash
cat logs/course-error.log
```

### Следить за access log

```bash
tail -f logs/course-access.log
```

### Найти `.log`-файлы внутри контейнера

```bash
docker compose exec nginx sh -c \
  'find /var/log/nginx -type f -name "*.log"'
```

### Найти логи по всему контейнеру

```bash
docker compose exec nginx sh -c \
  'find / -type f -name "*.log" 2>/dev/null'
```

`2>/dev/null` скрывает сообщения об отсутствии прав доступа.

---

## 9. Диагностика bind mounts

### Посмотреть все mounts контейнера

```bash
docker inspect nginx-course \
  --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

Пример:

```text
D:\...\html  -> /usr/share/nginx/html
D:\...\logs  -> /var/log/nginx
D:\...\geoip -> /usr/share/GeoIP
```

### Посмотреть реальный файл внутри временного контейнера

```bash
docker compose run --rm --entrypoint sh nginx -c \
  'nl -ba /etc/nginx/nginx.conf | sed -n "1,40p"'
```

Назначение:

- `--entrypoint sh` — вместо стандартного запуска Nginx открыть shell;
- `nl -ba` — вывести файл с номерами строк;
- `sed` — показать только нужный диапазон.

Это помогало находить случаи, когда `nginx.conf` случайно начинался сразу с `http {}` или `default.conf` — с `location`.

---

## 10. Проверка Linux-дистрибутива контейнера

В Git Bash прямой путь `/etc/os-release` может ошибочно преобразоваться в Windows-путь.

### Безопасный вариант

```bash
docker compose exec nginx sh -c 'cat /etc/os-release'
```

### Отключить path conversion Git Bash

```bash
MSYS_NO_PATHCONV=1 docker compose exec nginx cat /etc/os-release
```

`MSYS_NO_PATHCONV=1` запрещает Git Bash преобразовывать Linux-пути в Windows-пути.

---

## 11. Создание собственного Docker-образа

### Простой Dockerfile с пакетами

```dockerfile
FROM nginx:stable

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential \
        ca-certificates \
        curl \
        git \
        libmaxminddb0 \
        libmaxminddb-dev \
        mmdb-bin \
    && rm -rf /var/lib/apt/lists/*
```

Назначение пакетов:

| Пакет | Назначение |
|---|---|
| `build-essential` | `gcc`, `g++`, `make` и инструменты сборки |
| `ca-certificates` | проверка HTTPS-сертификатов при скачивании |
| `curl` | скачивание исходников |
| `git` | клонирование исходников GeoIP2-модуля |
| `libmaxminddb0` | runtime-библиотека MaxMind DB |
| `libmaxminddb-dev` | headers для компиляции GeoIP2-модуля |
| `mmdb-bin` | утилита `mmdblookup` |

### Указать Compose, что образ нужно собирать

```yaml
build:
  context: ./nginx
  dockerfile: Dockerfile
```

`context` определяет директорию, содержимое которой доступно во время сборки.

### Собрать образ

```bash
docker compose build nginx
```

### Чистая пересборка без кеша

```bash
docker compose build --no-cache nginx
```

Нужно, если Docker использует старые слои или необходимо заново выполнить установку пакетов.

### Подробный вывод сборки

```bash
docker compose build --no-cache --progress=plain nginx
```

Полезно для диагностики ошибок `configure` и `make`.

---

## 12. Multi-stage сборка GeoIP2-модуля

### Основные команды внутри Dockerfile

```dockerfile
RUN git clone --depth 1 \
    https://github.com/leev/ngx_http_geoip2_module.git
```

- клонирует исходники модуля;
- `--depth 1` скачивает только последний snapshot истории.

```dockerfile
RUN curl -fsSL \
    https://nginx.org/download/nginx-1.30.4.tar.gz \
    -o nginx.tar.gz
```

Флаги:

- `-f` — завершиться ошибкой при HTTP-ошибке;
- `-s` — не показывать progress bar;
- `-S` — всё же показать ошибку;
- `-L` — следовать redirect;
- `-o` — сохранить в файл.

```dockerfile
RUN tar -xzf nginx.tar.gz
```

Распаковывает `.tar.gz`:

- `x` — extract;
- `z` — gzip;
- `f` — имя архива передано далее.

```dockerfile
RUN ./configure \
    --with-compat \
    --add-dynamic-module=/build/ngx_http_geoip2_module
```

Подготавливает сборку:

- `--with-compat` — совместимость динамического модуля;
- `--add-dynamic-module` — собрать GeoIP2 как `.so`, а не встраивать его статически.

```dockerfile
RUN make modules
```

Компилирует только динамические модули, а не весь Nginx.

```dockerfile
COPY --from=builder \
    /build/nginx-1.30.4/objs/ngx_http_geoip2_module.so \
    /usr/lib/nginx/modules/ngx_http_geoip2_module.so
```

Копирует готовый `.so` из build-stage в минимальный финальный образ.

---

## 13. Проверка GeoIP2-модуля

### Проверить наличие файла

```bash
docker compose exec nginx sh -c \
  'ls -l /usr/lib/nginx/modules/ngx_http_geoip2_module.so'
```

Когда основной контейнер не запускается:

```bash
docker compose run --rm --entrypoint sh nginx -c \
  'ls -l /usr/lib/nginx/modules/ngx_http_geoip2_module.so'
```

### Проверить динамические зависимости

```bash
docker compose exec nginx sh -c \
  'ldd /usr/lib/nginx/modules/ngx_http_geoip2_module.so'
```

`ldd` показывает библиотеки, необходимые `.so`-файлу.

Ожидаемая зависимость:

```text
libmaxminddb.so.0 => /lib/.../libmaxminddb.so.0
```

Не должно быть:

```text
not found
```

### Проверить установленные MaxMind-пакеты

```bash
docker compose exec nginx dpkg -l | grep maxmind
```

### Проверить production-образ

```bash
docker compose exec nginx sh -c \
  'dpkg -l | grep -E "build-essential|libmaxminddb|mmdb-bin"'
```

В финальном multi-stage образе должны остаться runtime-пакеты, но не компилятор и development headers.

### Проверить `mmdblookup`

```bash
docker compose exec nginx which mmdblookup
```

```bash
docker compose exec nginx mmdblookup --help
```

---

## 14. Подключение базы GeoLite2

### Создать папку

```bash
mkdir -p geoip
```

### Проверить скачанный файл

```bash
ls -lh geoip
```

Ожидается:

```text
GeoLite2-Country.mmdb
```

### Подключить базу в Compose

```yaml
volumes:
  - ./geoip:/usr/share/GeoIP:ro
```

### Проверить базу внутри контейнера

```bash
docker compose exec nginx sh -c \
  'ls -lh /usr/share/GeoIP/GeoLite2-Country.mmdb'
```

### Выполнить поиск страны по IP

```bash
docker compose exec nginx sh -c '
  mmdblookup \
    --file /usr/share/GeoIP/GeoLite2-Country.mmdb \
    --ip 8.8.8.8 \
    country iso_code
'
```

Назначение:

- `--file` — путь к базе;
- `--ip` — адрес для поиска;
- `country iso_code` — путь к нужному полю записи.

Ожидаемый результат:

```text
"US" <utf8_string>
```

---

## 15. Проверка модулей Nginx

### Полная информация о сборке

```bash
docker compose exec nginx nginx -V
```

Показывает:

- версию Nginx;
- версию OpenSSL;
- все configure-аргументы;
- встроенные модули.

### Проверить Real IP module

```bash
docker compose exec nginx nginx -V 2>&1 | grep http_realip
```

Ищем:

```text
--with-http_realip_module
```

### Проверить HTTP/2 module

```bash
docker compose exec nginx nginx -V 2>&1 | grep http_v2
```

Ищем:

```text
--with-http_v2_module
```

`2>&1` объединяет stderr и stdout, потому что `nginx -V` пишет часть вывода в stderr.

---

## 16. Тестирование GeoIP через HTTP-заголовки

### Проверить GeoIP endpoint

```bash
curl \
  -H "X-Forwarded-For: 8.8.8.8" \
  http://localhost:8080/geo
```

- `-H` добавляет HTTP-заголовок;
- заголовок имитирует IP клиента для локального теста.

### Проверить HTTP-статус и заголовки

```bash
curl -i \
  -H "X-Forwarded-For: 8.8.8.8" \
  http://localhost:8080/restricted
```

`-i` выводит HTTP response headers вместе с телом.

Для заблокированной страны ожидается:

```text
HTTP/1.1 403 Forbidden
```

> В production нельзя доверять `X-Forwarded-For` от любого клиента.  
> `set_real_ip_from` должен содержать только доверенные proxy/CDN/load balancer адреса.

---

## 17. Создание локального TLS-сертификата

В Git Bash `/CN=localhost` воспринимается как Unix-путь и может преобразоваться в путь Windows.

### Корректная команда для Git Bash

```bash
MSYS_NO_PATHCONV=1 openssl req \
  -x509 \
  -nodes \
  -newkey rsa:2048 \
  -keyout certs/localhost.key \
  -out certs/localhost.crt \
  -days 365 \
  -subj "/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"
```

Разбор:

- `openssl req` — создать запрос/сертификат;
- `-x509` — сразу создать self-signed сертификат;
- `-nodes` — не шифровать приватный ключ паролем;
- `-newkey rsa:2048` — создать новый RSA-ключ длиной 2048 бит;
- `-keyout` — путь приватного ключа;
- `-out` — путь сертификата;
- `-days 365` — срок действия;
- `-subj` — Subject сертификата;
- `-addext` — добавить Subject Alternative Name.

### Удалить неудачно созданный ключ

```bash
rm -f certs/localhost.key
```

### Проверить созданные файлы

```bash
ls -lh certs
```

### Посмотреть сертификат

```bash
openssl x509 \
  -in certs/localhost.crt \
  -text \
  -noout
```

### Короткая информация о сертификате

```bash
openssl x509 \
  -in certs/localhost.crt \
  -noout \
  -subject \
  -issuer \
  -dates
```

---

## 18. Подключение HTTPS-порта и сертификатов

В `compose.yml`:

```yaml
ports:
  - "8080:80"
  - "8443:443"

volumes:
  - ./certs:/etc/nginx/certs:ro
```

Схема:

```text
http://localhost:8080  -> container:80
https://localhost:8443 -> container:443
```

### Проверить сертификаты внутри временного контейнера

```bash
docker compose run --rm --entrypoint sh nginx -c \
  'ls -lh /etc/nginx/certs'
```

---

## 19. Проверка HTTPS и HTTP/2

### Проверить HTTPS с self-signed сертификатом

```bash
curl -k https://localhost:8443/health
```

`-k` отключает проверку доверия сертификату. Допустимо только для локального теста.

### Попытаться использовать HTTP/2

```bash
curl -k --http2 -i https://localhost:8443/health
```

Если локальный `curl` собран без HTTP/2:

```text
the installed libcurl version does not support this
```

### Проверить возможности curl

```bash
curl --version
```

В `Features` ищем:

```text
HTTP2
```

### Проверить HTTP/2 временным Docker-контейнером

```bash
docker run --rm \
  curlimages/curl:latest \
  -k --http2 -i \
  https://host.docker.internal:8443/health
```

Почему используется `host.docker.internal`:

```text
localhost внутри временного контейнера
= сам временный контейнер

host.docker.internal
= Windows-хост с опубликованным портом 8443
```

### Подробная проверка ALPN

```bash
docker run --rm \
  curlimages/curl:latest \
  -k --http2 -v \
  https://host.docker.internal:8443/health
```

Ищем:

```text
ALPN: server accepted h2
```

---

## 20. Создание тестовых статических ресурсов

### Создать директории

```bash
mkdir -p html/assets/images
```

### Проверить MIME type CSS

```bash
curl -I http://localhost:8080/assets/style.css
```

`-I` отправляет HEAD-запрос и выводит только заголовки.

Ожидается:

```text
Content-Type: text/css
```

### Проверить MIME type SVG

```bash
curl -I http://localhost:8080/assets/images/image-1.svg
```

Ожидается:

```text
Content-Type: image/svg+xml
```

---

## 21. Полезные диагностические команды

### Показать последние 100 строк логов

```bash
docker compose logs --tail=100 nginx
```

### Запустить Compose на переднем плане

```bash
docker compose up
```

Удобно, когда контейнер падает: ошибка сразу остаётся в терминале.

### Временно отключить restart-loop

В `compose.yml`:

```yaml
restart: "no"
```

После этого контейнер не будет бесконечно перезапускаться при ошибке конфигурации.

### Проверить конкретные директивы активного конфига

```bash
docker compose exec nginx nginx -T 2>&1 \
  | grep -E "load_module|geoip2|http2 on|listen 443"
```

### Проверить доступность endpoint

```bash
curl http://localhost:8080/health
```

```bash
curl -k https://localhost:8443/health
```

---

# Итоговая структура проекта

```text
nginx-course-udemy/
├── certs/
│   ├── localhost.crt
│   └── localhost.key
├── geoip/
│   └── GeoLite2-Country.mmdb
├── html/
│   ├── index.html
│   └── assets/
├── logs/
├── nginx/
│   ├── Dockerfile
│   ├── nginx.conf
│   └── default.conf
└── compose.yml
```

# Рекомендуемый ежедневный workflow

После изменения конфигурации:

```bash
docker compose run --rm nginx nginx -t
```

Если проверка успешна:

```bash
docker compose up -d --force-recreate
```

Проверить состояние:

```bash
docker compose ps
```

Посмотреть ошибки:

```bash
docker compose logs --tail=100 nginx
```

После изменения только Nginx-конфига в работающем контейнере можно использовать:

```bash
docker compose exec nginx nginx -t &&
docker compose exec nginx nginx -s reload
```

# Важные правила

1. Не редактировать файлы вручную внутри контейнера — изменения исчезнут после пересоздания.
2. Конфигурацию и данные хранить в проекте и подключать через bind mounts.
3. Всегда выполнять `nginx -t` перед reload или restart.
4. Не использовать `:ro` для директории логов.
5. Не коммитить приватные ключи, license keys и секреты.
6. Не доверять `X-Forwarded-For` от всех источников.
7. Фиксировать версию Nginx при сборке динамических модулей.
8. Использовать multi-stage build, чтобы компилятор не попадал в production-образ.
