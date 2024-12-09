# **Изучение Postgresql**

<image src="./images/postgresql-cover.png" alt="postgresql-cover">

### **Задача:**

1. установи postgresql 12.2 и подключись к БД под пользователем postgres.  

2. Создай базу application и пользователя student с паролем pass, обладающего всеми правами на указанную базу.  

3. Создай новую последовательность task_id и таблицу tasks (в базе application), в которой поле id будет целочисленным значением, постоянно увеличивающимся на 1 при создании новой записи. Также создай поле name, которое будет содержать символьные данные длиной до 64 символов.  

4. Запиши в таблицу tasks 2 записи, определяя только поле name ('first_task', 'second_task'). После получите все данные из указанной таблицы, команды и вывод предоставьте.  

5. Введи команду:

SELECT user, pid, client_addr, query, query_start, NOW() - query_start AS elapsed  
FROM pg_stat_activity  
WHERE query != '<IDLE>'  
ORDER BY elapsed DESC;  

Постарайся разобраться в ее назначении. Модифицируй ее таким способом, чтобы помимо указанных условий вам выводились лишь строки, в которых есть запись в client_addr. 

в ответ скинь все команды и вывод
опиши весь пройденный путь, с какими ошибками встречался, как решал.

### **Для решения задачи будут использоваться:**

- redos-MUROM-7.3.4-20231220.0-Everything-x86_64-DVD1
- postgresql 12.2

### 1. Устанавливаем postgresql 12.2 и подключаемся под пользователем postgres.

Я решил установить postgresql 12.2 из исходников. Для этого скачиваем архив `postgresql-12.20.tar.gz` с исходниками. Распаковываем его командой `tar -xzvf postgresql-12.20.tar.gz`.
Затем устанавливаем необходимые зависимости командой `yum install -y gcc make readline-devel zlib-devel wget`. После этого выполняем команды:
- `./configure --prefix=/var/lib/postgresql_12.2` - подготавливаем наш софт для сборки. Будем устанавливать в директорию `/var/lib/postgresql_12.2`
- `make world` - запускаем компиляцию из исходного кода. Параметр `world`позволяет установить всё включая документацию и расширения.
- `make install-world` - запускаем процесс установки.

После этого создаём пользователя postgres командой `useradd -m -s /bin/bash postgres`, создаём каталог основного кластера баз данных командой `mkdir -p /var/lib/postgresql_12.2/main` и делаем владельцем каталога /var/lib/postgresql_12.2 и его подкаталогов пользователя postgres командой `chown -R postgres:postgres /var/lib/postgresql_12.2/`.

Подключаемся к пользователю postgres командой `sudo su - postgres` и задаём переменные окружения для него:
- `echo "export PGDATA=/var/lib/postgresql_12.2/main" >> .bash_profile`
- `echo "PATH=/var/lib/postgresql_12.2/bin/:$PATH" >> .bash_profile`
- `. .bash_profile`

Инициализируем кластер:
- `initdb`

Всё прошло успешно  

<image src="./images/initdb.jpg" alt="initdb">

Создадим службу SystemD для кластера:

- `vim /etc/systemd/system/postgresql-12.2.service` - создаём файл службы.   

Записываем туда следующие данные:
```ini
[Unit]
Description=PostgreSQL 12.2 database server
After=network.target
[Service]
Type=forking
User=postgres
ExecStart=/var/lib/postgresql_12.2/bin/pg_ctl -D /var/lib/postgresql_12.2/main -l /var/lib/postgresql_12.2/main/postgresql.log start 
ExecStop=/var/lib/postgresql_12.2/bin/pg_ctl -D /var/lib/postgresql_12.2/main stop
ExecReload=/var/lib/postgresql_12.2/pg_ctl -D /var/lib/postgresql_12.2/main reload
PIDFile=/var/lib/postgresql_12.2/main/postmaster.pid
[Install]
WantedBy=multi-user.target
```

Применим настройки, добавим сервис в автозагрузку и запустим службу командами `systemctl daemon-reload`, `stemctl enable postgresql-12.2.service` и `stemctl enable postgresql-12.2.service`.

Проверим всё ли запустилось 

<image src="./images/status.jpg" alt="status">

Подключаемся к БД под пользователем postgres:

- `sudo su - postgres`

### 2. Создаем базу application и пользователя student с паролем pass, обладающего всеми правами на указанную базу.

Для выполнения этого пункта запускаем утилиту `psql` и выполняем следующие команды:

- `CREATE DATABASE application;`
- `CREATE USER student WITH PASSWORD 'pass';`
- `GRANT ALL PRIVILEGES ON DATABASE application TO student;`

Результат работы представлен ниже

<image src="./images/creatdb.jpg" alt="creatdb">

### 3. Создём новую последовательность task_id и таблицу tasks по заданию.

Подключаемся к базе `application` под пользователем `student`:

- `psql -U student -d application`

Создаём последовательность `task_id`:

```sql
CREATE SEQUENCE task_id
START WITH 1 
INCREMENT BY 1
NO MINVALUE
NO MAXVALUE
CACHE 1;
```

Создаём таблицу `tasks` с полями `id` и `name`:

```sql
CREATE TABLE tasks (
    id INT DEFAULT NEXTVAL('task_id') PRIMARY KEY,
    name VARCHAR(64) NOT NULL
);
```

Результат работы представлен ниже

<image src="./images/sequence.jpg" alt="sequence">

### 4. Работаем с таблицей tasks по заданию.

Добавляем записи в таблицу `tasks`:

```sql
INSERT INTO tasks (name) VALUES ('first_task');
INSERT INTO tasks (name) VALUES ('second_task');
```

Все записи из таблицы можно получить командой `SELECT * FROM tasks;`.

Результат работы представлен ниже

<image src="./images/table_tasks.jpg" alt="table_tasks">

### 5. Разбираем по частям команду.

Есть следующая команда:

```sql
SELECT user, pid, client_addr, query, query_start, NOW() - query_start AS elapsed
FROM pg_stat_activity
WHERE query != '<IDLE>'
ORDER BY elapsed DESC;
```
После ввода команды получаем следующий результат:

<image src="./images/comand.jpg" alt="comand">

Эта команда выполняет запрос к системной таблице pg_stat_activity, которая содержит информацию обо всех активных сессиях в postgresql. При этом фильтр `WHERE` исключает записи, где запрос находится в состоянии `<IDLE>` (т.е. когда клиен подключён, но не выполняет запросы). А команда `ORDER BY elapsed DESC;` сортирует результат по времени выполнения запросов по убыванию.

Этот запрос может быть полезен если нужно провести диагностику производительности базы данных и мониторинга текущей активности базы данных.

Чтобы эта команда выводила только строки в которых есть запись в `client_addr` необходимо модифицировать фильтр запроса следующим образом:

```sql
SELECT user, pid, client_addr, query, query_start, NOW() - query_start AS elapsed
FROM pg_stat_activity
WHERE query != '<IDLE>' AND client_addr IS NOT NULL
ORDER BY elapsed DESC;
```

При этом, если `client_addr` не обозначен, то такая запись выводиться не будет.

После ввода такой команды у меня получился следующий результат:

<image src="./images/comand_mod.jpg" alt="comand_mod">