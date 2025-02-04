# Инструкция по развертыванию отказоустойчевого кластера PostgreSQL из двух узлов

### Версию PostgreSQL в данном примере я выбрал PostgrePro standart 11 так как в дальнейшем планирую крутить на нем базы 1С, и версию OS AstraLinux SE 1.7, но данная настройка подходит и для других версий, отличаются только некоторые команды и пути.
<br>

### Предполагается что есть два заранее подготовленых хоста, которые видят друг друга по IP и имени, у них заранее настроен ntp сервер, установлены правильные дата и время, установлена локаль utf-8 ru-RU по умолчанию и en-US дополнительно и т.п. <br> В моем примере настроены следующие две ноды:<br>1) pg1 - 10.192.168.101/24<br>2) pg2 - 10.192.168.102/24<br><br>Для общего ip (master-ip) будет присвоен адрес 10.192.168.103/24<br>Для создания двухузлового кластера будет использовано ПО "Pacemaker"
<br><hr><br>

## 1. Установка Postgre и настройка репликации.
1.1. На всех нодах - устанавливаем postgrespro-std-11:
```bash
apt install wget -y
cd /tmp
wget https://repo.postgrespro.ru/std-11/keys/pgpro-repo-add.sh
sh pgpro-repo-add.sh
apt-get install postgrespro-std-11 -y
```
<br>

1.2. На всех нодах - cоздаем архив:

```bash
su - postgres
mkdir /var/lib/pgpro/std-11/pg_archive
```
<br>

1.3. На ноде №1 - редактируем postgresql.conf, меняем следующие параметры:
```sql
listen_addresses = '*'
wal_level = hot_standby
synchronous_commit = on
archive_mode = on
archive_command = 'cp %p /var/lib/pgpro/std-11/pg_archive/%f'
max_wal_senders = 5
wal_keep_segments = 32
hot_standby = on
restart_after_crash = off
wal_receiver_status_interval = 2
max_standby_streaming_delay = -1
max_standby_archive_delay = -1
hot_standby_feedback = on
lc_messages = 'en_US.UTF-8'
```

1.4. На ноде №1 - редактируем pg_hba.conf, добавляем следующие параметры:
```sql
host    all             all             all                     md5
host    replication     all             10.192.168.0/24         trust
```

1.5. На ноде №1 - перезапускаем службу postgresql и добавляем пользователя с правами админа в СУБД:

```bash
systemctl restart postgrespro-std-11.service
su - postgres
psql
```
```sql
CREATE USER sa WITH SUPERUSER PASSWORD '123';
```
1.6. На ноде №2 - копируем данные с ноды №1:
```bash
su - postgres
rm -rf /var/lib/pgpro/std-11/data/*
pg_basebackup -h 10.192.168.101 -U postgres -D /var/lib/pgpro/std-11/data -X stream -P
```

1.7. На ноде №2 - создаем файл "/var/lib/pgpro/std-11/data/recovery.conf" для подтверждения репликации и добавляем туда строки ниже:
```bash
su - postgres
nano /var/lib/pgpro/std-11/data/recovery.conf
``` 
```sql
standby_mode = 'on'
primary_conninfo = 'host=10.192.168.101 port=5432 user=postgres application_name=pg2'
restore_command = 'cp /var/lib/pgpro/std-11/pg_archive/%f %p'
recovery_target_timeline = 'latest'
```

1.8. На ноде №2 - перезапускаем службу postgresql:

```bash
systemctl restart postgrespro-std-11.service
```

1.9. На ноде №1 - проверяем репликацию:

```bash
su - postgres
psql -c "select client_addr,sync_state from pg_stat_replication;"
```
если вывод выглядит так, то репликация настроена успешно:
```sql
  client_addr   | sync_state
----------------+------------
 10.192.168.102 | async
(1 строка)
```
1.10. На всех нодах - отключаем службу postgresql и переходим к настройке:
```bash
systemctl stop postgrespro-std-11.service
```
<br>
<br>

## 2. Развертывание двухузлового кластера pacemaker
2.1. На всех нодах - Устанавливаем pacemaker и меняем пароль пользователя hacluster в дальнейшем будем использовать `<пароль>`:
```bash
apt install pacemaker pcs -y
passwd hacluster
```
2.2. На ноде №1 - Инициализируем кластер:
```bash
pcs cluster destroy
```
2.3. На ноде №1 - Собираем кластер:
```bash
pcs host auth pg1 pg2 -u hacluster -p <пароль>
pcs cluster setup PG-Cluster pg1 pg2 --force
```
2.4. На ноде №1 - Запустим кластер:
```bash
pcs cluster start --all
```
2.5. На ноде №1 - Убедиться, что кластер полностью запустился:
```bash
while ! sudo pcs status ; do sleep 1 ; done
```
2.6. На ноде №1 - Отключить технологию STONITH и quorum:
```bash
pcs property set stonith-enabled=false
pcs property set no-quorum-policy=ignore
```
2.7. На ноде №1 - Проверим состояние ктастера (Если вывод `Online: [pg1 pg2]`) то кластер успешно стартовал:
```bash
pcs status
```
2.8. На ноде №1 - Проверим правильность конфигурации кластера командой:
```bash
crm_verify -L
```
2.9. На ноде №1 - Настроим поведение ресурсов:
```bash
pcs resource op defaults timeout=240s
```
2.10. На ноде №1 - Добавить IP-адрес кластера 192.168.23.100 как ресурс ClusterIP:
```bash
pcs resource create ClusterIP ocf:heartbeat:IPaddr2 ip=10.192.168.103 cidr_netmask=24 op monitor interval=30s
```
2.11. На ноде №1 - Создаем ресурс для PostgreSQL:

```bash
pcs resource create pgsql ocf:heartbeat:pgsql \
 pgctl="/opt/pgpro/std-11/bin/pg_ctl" \
 psql="/usr/bin/psql" \
 pgdata="/var/lib/pgpro/std-11/data/" \
 config="/var/lib/pgpro/std-11/data/postgresql.conf" \
 rep_mode="sync" \
 node_list="pg1 pg2" \
 restore_command="cp /var/lib/pgpro/std-11/pg_archive/%f %p" \
 primary_conninfo_opt="keepalives_idle=60 keepalives_interval=5 keepalives_count=5" \
 master_ip="10.192.168.103" \
 restart_on_promote='true' \
 op start timeout="60s" interval="0s" on-fail="restart" \
 op monitor timeout="60s" interval="4s" on-fail="restart" \
 op monitor timeout="60s" interval="3s" on-fail="restart" role="Master" \
 op promote timeout="60s" interval="0s" on-fail="restart" \
 op demote timeout="60s" interval="0s" on-fail="stop" \
 op stop timeout="60s" interval="0s" on-fail="block" \
 op notify timeout="60s" interval="0s"
 ```
 2.12. На ноде №1 - Установим параметры службы pgsql:
 ```bash
 pcs resource promotable pgsql promoted-max=1 promoted-node-max=1 clone-max=2 clone-node-max=1 notify=true
 pcs constraint colocation add ClusterIP with master pgsql-clone INFINITY
 ```
2.13. На всех нодах - Выполняем перезапуск
```bash
reboot
```
 2.14. На ноде №1 - Запустить кластер:
 ```bash
 pcs cluster start --all
 ```
 2.15. На ноде №1 - Проверяем что все завелось:
 ```bash
 pcs status
 ```


### 3. Восстановление в случае отказа
Для восстановления кластера в случае падения одной из нод требуется скопировать с мастера все данные на упавшую ноду, удалить файл блокировки, запустить кластер на упавшей ноде и очистить журнал ресурса. В моем примере я покажу случай, когда упала нода №1 (pg1) и мастером стала нода №2 (pg2), но команды одинаковы для любой из них.

```bash
su - postgres
rm -rf /var/lib/pgpro/std-11/data/*
pg_basebackup -h 10.192.168.103 -U postgres -D /var/lib/pgpro/std-11/data -X stream -P
exit
rm /var/lib/pgsql/tmp/PGSQL.lock
pcs cluster start
pcs resource cleanup ClusterIP
pcs resource cleanup pgsql
```
Далее выполняем `pcs status` чтобы убедиться что pg1 поднялся и занял статус slave.