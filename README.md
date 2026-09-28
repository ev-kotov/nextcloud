# nextcloud

Nextcloud на базе Rootless Podman и Mise

Безопасное и изолированное частное облако, развернутое без root-привилегий.

## Сценарии обслуживания и развертывания

### Сценарий 1. Чистая установка с нуля

Используйте этот метод только при первоначальной установке или если нужно полностью обнулить инстанс:

1. **Остановите старые контейнеры и сотрите именованные тома базы данных в Podman:**

    ```bash
    mise run prod:down
    podman volume rm $(podman volume ls -q | grep nextcloud) 2>/dev/null || true
    ```

2. **Физически очистите локальные папки на диске хоста:**

    ```bash
    sudo rm -rf /srv/nextcloud/data
    sudo rm -rf /opt/stacks/nextcloud/app/
    sudo mkdir -p /srv/nextcloud/data
    ```

3. **Запустите последовательную чистую инсталляцию:**

    ```bash
    mise run prepare
    mise run prod:up
    mise run prod:db-init
    mise run prod:db-configure
    ```

### Сценарий 2. Развертывание на имеющихся данных (Перезапуск/Обновление)

Если вам нужно обновить код приложения, пересоздать контейнеры или перенести файлы на этот сервер из бэкапа (база данных в томах Podman и файлы в `/srv/nextcloud/data` остаются нетронутыми):

1. **Безопасно очистите только контейнеры и временный кэш кода:**

    ```bash
    mise run prod:clean
    ```

2. **Убедитесь, что права и контексты SELinux на хосте соответствуют требованиям:**

    ```bash
    mise run prepare
    ```

3. **Поднимите стек:**

    ```bash
    mise run prod:up
    ```

    *Контейнер Nextcloud запустится, обнаружит существующий `nextcloud/app/config.php` и ваши файлы, после чего сразу продолжит работу в штатном режиме без переинициализации таблиц.*

## Перенос данных (миграция на новый сервер)

Для переноса работающего инстанса Nextcloud на другую машину выполните три шага:

1. **Резервная копия базы данных:** Сделайте горячий дамп PostgreSQL на старом сервере:

    ```bash
    podman exec nextcloud-db pg_dump -U postgres nextcloud_db > nextcloud_db.sql
    ```

2. **Копирование файлов:** Перенесите директорию данных `/srv/nextcloud/data` и конфигурации `/opt/stacks/nextcloud/app/config` на новый хост с помощью `rsync` для сохранения оригинальных владельцев (`rsync -a`).

3. **Восстановление схемы СУБД:** Поднимите новый стек (`mise run prepare && mise run prod:up`), создайте чистую базу (`mise run prod:db-init` — это создаст пользователя и структуру), а затем импортируйте старый дамп поверх:

    ```bash
    cat nextcloud_db.sql | podman exec -i nextcloud-db psql -U postgres -d nextcloud_db
    ```
