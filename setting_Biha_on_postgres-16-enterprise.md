# Настройка Biha на postgres-16-enterprise



Инструкция:

1) Установка postgres pro enterprise (обязательно версия enterprise) из репозитория
2) Инициализируем кластер biha на ведущем сервере:

Сначала изолируем сервер postgresql от нагрузки, т.е. выключаем все сервисы, которые могут к нему обращаться. В нашем случае мы выключаем сервер 1С.
Далее мы выключаем сервер postgresql.
```bash
systemctl stop postgrespro-ent-16.service 
```



Выполняем вход под пользователем postgres.
```bash
su postgres
```



Конвертируем текущий кластер postgres в кластер biha:
```bash
postgres@dbserver1:~$ init —-convert —-biha-node-id=1 —-host=192.168.0.100 --port=5432 --biha-port=5433 —-nquorum=2 —-minnodes=2 —-pgdata=/var/lib/pgpro/ent-16/data 
```



- вводим пароль для `biha_replication_user` и копируем выданный ключ (обязательно сохраняем)

- Далее вводим команду: `echo *:*:*:biha_replication_user:12345678 >> /var/lib/postgresql/.pgpass` - добавляем пароль ранее введёный для пользователя в `pgpass`

```bash
postgres@dbserver1:~$ echo *:*:*:biha_replication_user:12345678 >> /var/lib/postgresql/.pgpass
```



- Ограничиваем доступ к файлу паролей postgres.
```bash
postgres@dbserver1:~$ chmod 600 /var/lib/postgresql/.pgpass 
```



3) Настройка методов аутентификации.
Biha при создании кластера редактирует файлы настроек кластера, а также добавляет свои файлы настроек. Одним из файлов настроек biha будет `/var/lib/pgpro/ent-16/data/pg_hba.biha.conf`  , и его нам нужно будет отредактировать.
Не все платформы 1С умеют работать в режиме аутентификации scram-sha-256, поэтому рекомендуем переключить методы аутентификации на md5.

4) Выходим из пользователя postgres (ctrl + d) и запускаем службу postgres.
```bash
systemctl start postgrespro-ent-16.service 
```



Обратно входим в пользователя postgres и проверяем статус кластера:
```bash
su — postgres 
postgres@dbserver1:~$ bihactl status —host=localhost —port=5432 
```
И выходим из postgres


5) Добавление ведомого узла в кластер (выполняем на втором узле):
```bash
apt install postgrespro-ent-16 
systemctl stop postgrespro-ent-16.service 
rm -r /var/lib/pgpro/ent-16/data 
```



Вводим команду (выполняем на первом узле): `echo *:*:*:biha_replication_user:12345678 >> /var/lib/postgresql/.pgpass`  - добавляем пароль ранее введёный для пользователя в pgpass
```bash
su — postgres
postgres@dbserver1:~$ echo *:*:*:biha_replication_user:12345678 >> /var/lib/postgresql/.pgpass
```



- Ограничиваем доступ к файлу паролей postgres.
```bash
postgres@dbserver1:~$ chmod 600 /var/lib/postgresql/.pgpass 
```



Добавляем второй узел (выполняем на втором узле) в кластер biha из под учётной записи postgres, так же назначаем pgpass на пользовтеля postgres:postgres
```bash
postgres@dbserver2:~$  bihactl add --biha-node-id=2 --host=192.168.0.101 --port=5432 --biha-port=5433 --magic-string='ключ который оставляли ранее=' --pgdata=/var/lib/pgpro/ent-16/data 
```
Тут мы также как и на ведомом сервере для параметра host указываем именно IP адрес ведомого сервера, а не его имя. И для параметра magic-string указываем текст строки, который нам вернулся при создании кластера на ведущем сервере.

Добавляем в кластер 3 -ий сервер выступающий в роли судьи:
```bash
root@dbserver2:~$ apt install postgrespro-ent-16 
root@dbserver2:~$ systemctl stop postgrespro-ent-16.service 
root@dbserver2:~$ rm -r /var/lib/pgpro/ent-16/data 
```



Вводим команду: `echo *:*:*:biha_replication_user:12345678 >> /var/lib/postgresql/.pgpass`  - добавляем пароль ранее введёный для пользователя в pgpass
```bash
root@dbserver1:~$ su — postgres
postgres@dbserver1:~$ echo *:*:*:biha_replication_user:12345678 >> /var/lib/postgresql/.pgpass
```



Ограничиваем доступ к файлу паролей postgres.
```bash
postgres@dbserver1:~$ chmod 600 /var/lib/postgresql/.pgpass
```



Добавляем торетий хост в роли слушателя

```bash
bihactl add --biha-node-id=3 --host=192.168.0.102--port=5432 --biha-port=5433 --magic-string='наш ключ' --pgdata=/var/lib/pgpro/ent-16/data --mode=referee_with_wal 
```

Установка keepalive на 2-ух ВМ но не на рефери ВМ:
```bash
apt install keepalived
```
 

Настройка keepalived( `/etc/keepalived/keepalived.conf`  ):
```bash
global_defs {
    enable_script_security
}

vrrp_script chk_pg_status {
    script "/etc/keepalived/check_pg_status.sh"
    interval 5
    user root
}

vrrp_instance VI_1 {
    state MASTER
    interface enp0s3
    virtual_router_id 254
    priority 100
    advert_int 2
    virtual_ipaddress {
        192.168.0.150/24
    }


    track_script {
chk_pg_status
    }
}
```



Скрипт ( `/etc/keepalived/check_pg_status.sh`  ) id меняем в зависимости от id node:
```bash
#!/bin/bash

# Проверка состояния
ROLE=$(sudo -u postgres psql -d biha_db -c "SELECT state FROM biha.status_v WHERE id = 2;" -t -A)
# Условие для выхода
if [[ "$ROLE" == 'LEADER_RW' ]]; then
    exit 0  # Узел является LEADER_RW
else
    exit 1  # Узел является FOLLOWER
fi
```



Не забыть выдать права на `/etc/keepalived/check_pg_status.sh`
```bash
chmod +x /etc/keepalived/check_pg_status.sh
chmod 750 /etc/keepalived/check_pg_status.sh
```




