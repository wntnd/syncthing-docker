# Syncthing + автоматический HTTPS

## Первый запуск

1. В `Caddyfile` замени `sync.example.com` и `admin@example.com` на свой домен
   и почту. Создай DNS A-запись на IP VPS. AAAA — только при рабочем IPv6.
   Если используешь Cloudflare, запись должна быть **DNS only**.

2. На VPS создай отдельного системного пользователя без возможности входа
   и каталоги для файлов:

   ```sh
   sudo useradd --system --user-group \
     --home-dir /srv/syncthing \
     --no-create-home \
     --shell /usr/sbin/nologin \
     syncthing
   sudo mkdir -p /srv/syncthing/config /srv/syncthing/data
   sudo chown syncthing:syncthing /srv/syncthing/config /srv/syncthing/data
   id syncthing
   ```

   Если пользователь `syncthing` уже существует, не создавай его повторно:
   проверь его назначение и права перед использованием.

   UID — числовой идентификатор пользователя, GID — группы. Linux проверяет
   права на примонтированные файлы по этим числам, причём внутри контейнера
   и на VPS они должны совпадать. Отдельный пользователь отделяет владельца
   файлов Syncthing от твоего обычного пользователя и не требует запуска от root.
   Устанавливать Syncthing на хост или разрешать этому пользователю вход не нужно.

   Например, если `id syncthing` вывел `uid=995(syncthing) gid=994(syncthing)`,
   замени `user: "1000:1000"` в `docker-compose.yaml` на:

   ```yaml
   user: "995:994"
   ```

   **Используй числа из своего вывода `id`, а не из примера.** Пользователь
   с таким именем внутри контейнера не нужен. Не используй `0:0` или `chmod 777`.
   Каталоги Caddy передавать пользователю `syncthing` не нужно.

   После этого запусти только Syncthing:

   ```sh
   docker compose up -d syncthing
   ```

   Эти команды рассчитаны на первоначальную настройку. Если в `config` и `data`
   уже есть файлы и ты меняешь UID/GID, сначала останови стек и проверь владельцев
   вложенных файлов: обычный `chown` выше меняет владельцев только самих каталогов.

3. **До запуска Caddy задай логин и пароль.** На своём компьютере открой туннель:

   ```sh
   ssh -N -L 9999:127.0.0.1:8384 USER@SERVER_IP
   ```

   Зайди на `http://127.0.0.1:9999` → Actions → Settings → GUI.
   Задай GUI Authentication User и длинный уникальный пароль, сохрани.
   Встроенный HTTPS Syncthing включать не нужно: им занимается Caddy.

4. На VPS запусти остальные сервисы:

   ```sh
   docker compose up -d
   ```

   Панель: `https://ТВОЙ-ДОМЕН`. Caddy сам получит и будет продлевать сертификат.
   **Не запускай Caddy раньше настройки пароля:** иначе панель будет публичной
   без авторизации. SSH-туннель после настройки можно закрыть.

## Сеть и синхронизация

В firewall сервера и провайдера разреши `80/TCP`, `443/TCP`, `22000/TCP`,
`22000/UDP`. Сохрани доступ по SSH. `8384` слушает только localhost VPS,
`21027/UDP` наружу не нужен. Учти, что публикация портов Docker может обходить UFW.

Прямое подключение TCP/QUIC Syncthing предпочитает **по умолчанию**. Relay
используется при невозможности прямого соединения; попытки соединиться напрямую
продолжаются и после подключения через relay. Ничего специально переключать не надо.
Не отключай relays и global discovery в настройках.

На клиентах адрес устройства сервера можно оставить `dynamic` либо указать:

```text
tcp://ТВОЙ-ДОМЕН:22000, quic://ТВОЙ-ДОМЕН:22000, dynamic
```

Для VPS можно выключить Local Discovery и NAT traversal в Settings → Connections:
при публичном IP и открытых портах они не нужны.

Добавляй папки с путями `/var/syncthing/data/ИМЯ` — на VPS это
`/srv/syncthing/data/ИМЯ`. Если появилась стандартная папка `/var/syncthing/Sync`,
удали её из списка папок или измени путь: она не входит в постоянный bind mount.

## Обслуживание

- Конфигурация, ключи и база: `/srv/syncthing/config`.
- Файлы: `/srv/syncthing/data`.
- Сертификаты Caddy: `/srv/syncthing/caddy`. Не удаляй их между перезапусками.
- Включи версионирование нужных папок и отдельный бэкап: синхронизация распространяет
  удаления и не заменяет резервную копию. Для согласованного бэкапа всей директории
  `/srv/syncthing` сначала останови стек через `docker compose down`.
- Версии закреплены в compose. Обновляй теги после проверки release notes и бэкапа,
  затем `docker compose pull` и `docker compose up -d`.
- Логи: `docker compose logs --tail=100 caddy syncthing`.

Документация: [Docker](https://github.com/syncthing/syncthing/blob/main/README-Docker.md),
[firewall](https://docs.syncthing.net/users/firewall.html),
[relay](https://docs.syncthing.net/users/relaying.html),
[HTTPS Caddy](https://caddyserver.com/docs/automatic-https).
