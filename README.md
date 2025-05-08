# Часть 1 
Подробная инструкция по настройке отказоустойчивого кластера PostgreSQL

## Практическую работу выполняли: 
- Беликова Анастасия Вячеславовна 
- Фоминых Дмитрий Алекеевич
- Группа РИ-411050

### Подготовительные шаги

1. Установка необходимого ПО
На всех узлах (Master, Replica и Arbiter) выполнить:


## Для Ubuntu/Debian
``` python
  sudo apt update
  sudo apt install -y postgresql-12 postgresql-client-12 python3 python3-pip
  sudo pip3 install psycopg2-binary
``` 

2. Настройка сетевых имен (опционально)
Добавьте в /etc/hosts на всех узлах:
``` python
192.168.1.1 pg-master
192.168.1.2 pg-replica
192.168.1.3 pg-arbiter
``` 
## Настройка Master-узла
1. Инициализация БД
``` python
sudo -u postgres /usr/lib/postgresql/12/bin/initdb -D /var/lib/postgresql/12/main
``` 
3. Настройка конфигурации
Отредактируйте /var/lib/postgresql/12/main/postgresql.conf:
``` python
ini
listen_addresses = '*'
wal_level = replica
max_wal_senders = 3
synchronous_commit = on
synchronous_standby_names = 'pg-replica'
``` 
3. Настройка доступа
Добавьте в /var/lib/postgresql/12/main/pg_hba.conf:
``` python
host    replication     replicator      pg-replica/32       md5
host    all             all             pg-replica/32       md5
``` 

4. Создание пользователя репликации
``` python
sudo -u postgres psql -c "CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'securepassword';"
``` 
7. Запуск сервера
``` python
sudo systemctl start postgresql@12-main
``` 
## Настройка Replica-узла

1. Остановка PostgreSQL (если запущен)
``` python 
sudo systemctl stop postgresql
``` 
3. Создание резервной копии с Master
``` python
sudo -u postgres rm -rf /var/lib/postgresql/12/main/*
sudo -u postgres pg_basebackup -h pg-master -U replicator -D /var/lib/postgresql/12/main -P -R -X stream
``` 
Введите пароль securepassword при запросе.

5. Настройка файла standby.signal
``` python
sudo -u postgres touch /var/lib/postgresql/12/main/standby.signal
``` 
7. Проверка конфигурации
Убедитесь, что в /var/lib/postgresql/12/main/postgresql.auto.conf есть:

ini
primary_conninfo = 'user=replicator password=securepassword host=pg-master port=5432 sslmode=prefer sslcompression=0 gssencmode=prefer krbsrvname=postgres target_session_attrs=any'

5. Запуск сервера
``` python
sudo systemctl start postgresql@12-main
``` 
## Настройка Arbiter-узла

1. Установка простого TCP-сервера
Создайте файл /opt/pg_arbiter.py:

```python
import socket
import threading

def handle_client(conn, addr):
    try:
        data = conn.recv(1024)
        if data == b"CHECK_MASTER":
            # Проверяем доступность мастера
            test_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            test_sock.settimeout(5)
            try:
                test_sock.connect(('pg-master', 5432))
                conn.sendall(b"MASTER_UP")
            except:
                conn.sendall(b"MASTER_DOWN")
            finally:
                test_sock.close()
    finally:
        conn.close()

def start_server():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.bind(('0.0.0.0', 5432))
        s.listen()
        while True:
            conn, addr = s.accept()
            threading.Thread(target=handle_client, args=(conn, addr)).start()

if __name__ == "__main__":
    start_server()
```

2. Запуск арбитра
``` python
sudo python3 /opt/pg_arbiter.py &
``` 
## Настройка агента мониторинга

1. Создание конфигурационных файлов
   
На Master (/etc/pg_agent.conf):

``` python
[node]
type = master
id = node1
host = pg-master

[postgres]
port = 5432
data_dir = /var/lib/postgresql/12/main

[arbiter]
host = pg-arbiter

[monitoring]
check_interval = 10
timeout = 5
На Replica (/etc/pg_agent.conf):
```
```ini
[node]
type = replica
id = node2
host = pg-replica

[postgres]
port = 5432
data_dir = /var/lib/postgresql/12/main

[replication]
master_host = pg-master

[arbiter]
host = pg-arbiter

[monitoring]
check_interval = 10
timeout = 5
``` 
2. Запуск агента
На всех узлах:

``` python
sudo curl -o /opt/postgres_agent.py https://raw.githubusercontent.com/your-repo/postgres-ha-agent/main/agent.py
sudo python3 /opt/postgres_agent.py /etc/pg_agent.conf
``` 
Проверка работы
1. Проверка репликации
На Master:

``` python
sudo -u postgres psql -c "SELECT * FROM pg_stat_replication;"
``` 
Должна быть видна реплика.

2. Тестирование отказоустойчивости
Остановите PostgreSQL на Master:

``` python
sudo systemctl stop postgresql@12-main
``` 
Через 10-15 секунд проверьте статус на Replica:

``` python
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
``` 
Должно вернуться f (false), что означает переход в режим master.

Проверьте запись данных на новом Master:
``` python
sudo -u postgres psql -c "CREATE TABLE test_failover(id serial PRIMARY KEY);"
``` 

## Автозапуск агента
Создайте systemd сервис (/etc/systemd/system/pg_agent.service):
``` ini
[Unit]
Description=PostgreSQL HA Agent
After=postgresql.service

[Service]
ExecStart=/usr/bin/python3 /opt/postgres_agent.py /etc/pg_agent.conf
Restart=always
User=postgres

[Install]
WantedBy=multi-user.target
Затем:


sudo systemctl daemon-reload
sudo systemctl enable pg_agent
sudo systemctl start pg_agent
``` 
