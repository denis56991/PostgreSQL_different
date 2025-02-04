# Разворачивание и обслуживание PostgreSQL (16) для 1С на примере Ubuntu 22.04

1) Настройка Ubuntu

После накатки ОС необходимо предварительно сменить локали системы (это оч важно, иначе постгрес раскатится в дефолтной локали en_US и при создании базы в 1с будут сыпать ошибки)
```bash
locale-gen en_US ru_RU en_US.UTF-8 ru_RU.UTF-8
export LANG="ru_RU.UTF-8"
update-locale LANG=ru_RU.UTF-8
reboot
```
Если получилось так, что постгрес все же установили до смены глобальных локалей в системе, дефолтный кластер нужно инициализировать заново таким образом:
```bash
initdb --locale=ru_RU -E UTF8 -D /var/lib/pgpro/1c-16/data
```
2) Процесс установки postgres из официального письма:

2.1) Установка с интернетом
```bash
wget https://repo.postgrespro.ru/1c-16/keys/pgpro-repo-add.sh
sh pgpro-repo-add.sh
```
Если наш продукт единственный Postgres на вашей машине и вы хотите
сразу получить готовую к употреблению базу:
```bash
apt-get install postgrespro-1c-16
```
Если у вас уже установлен другой Postgres и вы хотите чтобы он
продолжал работать параллельно (в том числе и для апгрейда с более
старой major-версии):
```bash
apt-get install postgrespro-1c-16-contrib
/opt/pgpro/1c-16/bin/pg-setup initdb
/opt/pgpro/1c-16/bin/pg-setup service enable
/opt/pgpro/1c-16/bin/pg-setup service start
```
Данная репа, судя по заверениям на сайте postgrespro, является пропатченной версией обычного постгреса, не требующей лицензирования (т.е. это не postgrespro standard/enterprise)

2.2) Установка БЕЗ интернета (ручная сборка из пакетов с зависимостями)

Если клиент находится в закрытом контуре, то понадобится для начала выгрузить postgresql с необходимыми зависимостями с машины с АНАЛОГИЧНОЙ ОС (важно).

Процедура следующая:

# Выгружаем пакет и зависимости и запаковываем в архив
```bash
apt-get download $(apt-rdepends postgrespro-1c-16 | grep -v "^ " | grep -v "^debconf") && tar -czvf packages.tar.gz *.deb
```
Далее, просим клиента загрузить архив `packages.tar.gz` на машину (например, в папку `/tmp`) и устанавливаем:
```bash
cd /tmp
tar -zxvf packages.tar.gz  
sudo dpkg -i *.deb
```
3) Первоначальная настройка
```bash
sudo -u postgres psql
```
Чекаем, что дефолтные базы создались в нужной локали

Меняем пароль postgres + при необходимости создаем сервисные учетки для бэкапирования/создания базы 1С с нужными правами
```sql
ALTER USER postgres WITH PASSWORD 'gigapass123';

/* dbreator 1C */
CREATE USER dbcreator with password 'megaPASS1';
ALTER USER dbcreator CREATEDB;
ALTER USER dbcreator SUPERUSER;

/* сервисная учетка dbbackup */
CREATE USER dbbackup WITH PASSWORD 'SUPERpass987';
GRANT pg_read_all_data TO dbbackup;
```
4) Доступы к postgresql

Закрываем доступы к постгресу всем, кроме 1С

```/var/lib/pgpro/1c-16/data/pg_hba.conf ```

тут в строке меняем `0.0.0.0/0` на айпишник сервера 1С
```conf
host            all             all             ip_1c/32       md5
```
```/var/lib/pgpro/1c-16/data/postgresql.conf```

в строке `listen_addresses` вместо `'*'` оставляем `localhost` (чтобы сам кластер мог запускаться) и айпишник 1с 

```conf
listen_addresses = 'ip_1c,localhost'
```

5) Рихтуем конфиги
Увеличение интервалов выполнения автовакуума

```/var/lib/pgpro/1c-16/data/postgresql.conf```

```conf
autovacuum_naptime = 10m

Подрубаем логирование постгреса

logging_collector = on 
```



6) Скрипты обслуживания баз (в примере две базы)

Скрипты храним в папках `/etc/cron.daily` `/etc/cron.weekly` без указания расширения (например: `pg_backup`, но не `pg_backup.sh`)

Крон для данных скриптов (регулярность запуска) редактируется в файле `/etc/crontab`

Раз в неделю запускаем полный Вакуум и Реиндексирование
```bash
#!/bin/bash

PATH=/etc:/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin

# Parameters for connecting to the PostgreSQL database
DB_USER="postgres"
DATABASE1="zup_prod"
DATABASE2="bf_prod"
TIMEOUT_SECONDS=36000  # 10 hours in seconds

# Function to execute vacuumdb and reindexdb with timeout
execute_vacuum_and_reindex() {
    local database="$1"

    echo "Executing VACUUM and REINDEX for database '$database'..."
    timeout $TIMEOUT_SECONDS vacuumdb --username=$DB_USER --dbname=$database --verbose --analyze --full || { echo "Execution timeout! Aborting script."; exit 1; }
    timeout $TIMEOUT_SECONDS reindexdb --username=$DB_USER --dbname=$database || { echo "Execution timeout! Aborting script."; exit 1; }

    # Check if timeout occurred
    if [ $? -eq 124 ]; then
        echo "Index reindexing execution timeout occurred. Aborting script."
        exit 1
    fi
}

# Set timeout handler for SIGTERM signal
timeout_handler() {
    echo "Execution time exceeded. Aborting script."
    killall -TERM vacuumdb reindexdb  # Send SIGTERM signal to all processes
    exit 1
}

# Set timeout handler for SIGTERM
trap 'timeout_handler' SIGTERM

# Execute vacuumdb and reindexdb for database1
execute_vacuum_and_reindex "$DATABASE1"

# Execute vacuumdb and reindexdb for database2
execute_vacuum_and_reindex "$DATABASE2"
```



Бэкапируем базы 1 раз в день, храним 5 дней (уточняем у клиента глубину бэкапа и количество точек в день!)
```bash
 #!/bin/bash

pathB=/media/pgpro/1c-16/backup
dbUser=dbbackup

# Function to create pg_dump for a database
create_pg_dump() {
    local db_name="$1"
    local dump_file="pg_dump_${db_name}_$(date +%Y%m%d_%H%M%S).sql.gz"
    /opt/pgpro/1c-16/bin/pg_dump -U $dbUser -d "$db_name" -Z 9 > "$pathB/$dump_file"
}

# Function to remove backups older than 5 days
remove_old_backups() {
    local backup_dir="$1"
    find "$backup_dir" -type f -mtime +5 -exec rm {} +
}

create_pg_dump "bf_prod"

create_pg_dump "zup_prod"

remove_old_backups "$pathB"
```
