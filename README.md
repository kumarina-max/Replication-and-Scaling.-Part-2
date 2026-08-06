
## Домашнее задание к занятию «Репликация и масштабирование. Часть 2» Марина Кукушкина

## Задание 1 

### 1. Активный master + пассивный slave 

Отказоустойчивость – при сбое мастера slave быстро берёт на себя нагрузку (failover), сокращая время простоя.

Резервное копирование без нагрузки – бэкапы можно делать с slave, не нагружая основной сервер.

Тестирование и отчёты – slave можно использовать для проверки изменений или генерации отчётов, не влияя на продуктивный мастер.

### 2. Master + несколько slave-серверов

Горизонтальное масштабирование чтения – запросы SELECT распределяются между несколькими slave, что увеличивает пропускную способность системы.

Повышенная надёжность – выход одного slave не влияет на доступность, нагрузка автоматически перераспределяется на остальные.

Географическая распределённость – slave можно размещать в разных дата-центрах для снижения задержек для удалённых пользователей.

Специализация – разные slave могут обслуживать разные типы задач (аналитика, поиск, отчёты), изолируя нагрузки.


## Задание 2

### Принципы построения

### 1. Вертикальный шардинг
Разделяем таблицы по функциональным доменам на отдельные БД:

БД users (пользователи)

БД books (книги)

БД shops (магазины)

2. Горизонтальный шардинг

Каждую доменную БД делим на несколько шард (например, по 4) по хешу/диапазону ключа:

users – по user_id

books – по book_id

shops – по shop_id (или региону)


 ## Архитектура системы
 
````mermaid
flowchart TD
    Client["Клиентские приложения"]
    Router["Шардинг-роутер<br>(по ключу: user_id / book_id / shop_id)"]

    subgraph UsersDB["БД Пользователей"]
        U["Шарды 1–4<br>(каждый: Master + Slave)"]
    end

    subgraph BooksDB["БД Книг"]
        B["Шарды 1–4<br>(каждый: Master + Slave)"]
    end

    subgraph ShopsDB["БД Магазинов"]
        S["Шарды 1–4<br>(каждый: Master + Slave)"]
    end

    Client --> Router
    Router -->|"user_id"| UsersDB
    Router -->|"book_id"| BooksDB
    Router -->|"shop_id"| ShopsDB
````

### Режимы работы серверов (на каждый шард)

Master – принимает запись (INSERT/UPDATE/DELETE), синхронно реплицирует изменения.

Slave(ы) – только чтение (SELECT), разгружают мастера. При сбое мастера – автоматический failover (slave становится мастером).

Все шарды работают независимо, кросс-шардные JOIN’ы вынесены на уровень приложения.

Таким образом вертикальное разделение изолирует нагрузки, горизонтальное – обеспечивает масштабируемость и отказоустойчивость.


## Задание 3

````
Структура проекта

sharding-demo/
├── docker-compose.yml
├── init/
│   ├── users/
│   │   └── init.sql
│   ├── books/
│   │   └── init.sql
│   └── shops/
│       └── init.sql
└── setup-fdw.sql
````
### docker-compose.yml

Описывает пять сервисов:

postgres-users – сервер для таблицы users (порт 5432)

postgres-books – сервер для таблицы books (порт 5433)

postgres-shops-master – мастер для таблицы shops (порт 5434)

postgres-shard1 – шард 1 (регион center, порт 5435)

postgres-shard2 – шард 2 (регион siberia, порт 5436)

````
version: '3.8'

services:
  postgres-users:
    image: postgres:15
    container_name: postgres-users
    environment:
      POSTGRES_PASSWORD: strongpassword
      POSTGRES_DB: users_db
    ports:
      - "5432:5432"
    volumes:
      - ./init/users:/docker-entrypoint-initdb.d
      - users_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  postgres-books:
    image: postgres:15
    container_name: postgres-books
    environment:
      POSTGRES_PASSWORD: strongpassword
      POSTGRES_DB: books_db
    ports:
      - "5433:5432"
    volumes:
      - ./init/books:/docker-entrypoint-initdb.d
      - books_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  postgres-shops-master:
    image: postgres:15
    container_name: postgres-shops-master
    environment:
      POSTGRES_PASSWORD: strongpassword
      POSTGRES_DB: shops_db
    ports:
      - "5434:5432"
    volumes:
      - ./init/shops:/docker-entrypoint-initdb.d
      - shops_master_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  postgres-shard1:
    image: postgres:15
    container_name: postgres-shard1
    environment:
      POSTGRES_PASSWORD: strongpassword
      POSTGRES_DB: shops_db
    ports:
      - "5435:5432"
    volumes:
      - ./init/shops:/docker-entrypoint-initdb.d
      - shard1_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  postgres-shard2:
    image: postgres:15
    container_name: postgres-shard2
    environment:
      POSTGRES_PASSWORD: strongpassword
      POSTGRES_DB: shops_db
    ports:
      - "5436:5432"
    volumes:
      - ./init/shops:/docker-entrypoint-initdb.d
      - shard2_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  users_data:
  books_data:
  shops_master_data:
  shard1_data:
  shard2_data:

````
## init/users/init.sql

````
CREATE TABLE IF NOT EXISTS users (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
````

## init/books/init.sql


````
CREATE TABLE IF NOT EXISTS books (
    book_id SERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    author VARCHAR(255),
    isbn VARCHAR(20) UNIQUE,
    price DECIMAL(10, 2),
    published_year INTEGER
);

````

## init/shops/init.sql

Этот скрипт выполняется на мастере и на обоих шардах, создавая одинаковую структуру таблицы.


````
CREATE TABLE IF NOT EXISTS shops (
    shop_id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    address TEXT,
    region VARCHAR(50) NOT NULL,
    phone VARCHAR(20)
);

````

## setup-fdw.sql
Этот скрипт выполняется вручную на мастере (postgres-shops-master) после запуска контейнеров. Он настраивает FDW, переименовывает таблицы, создаёт представление и триггер.

````
-- Подключение к БД shops_db
\c shops_db;

-- Включение расширения
CREATE EXTENSION IF NOT EXISTS postgres_fdw;

-- Создание серверов для шардов
CREATE SERVER IF NOT EXISTS shard1_server
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'postgres-shard1', port '5432', dbname 'shops_db');

CREATE SERVER IF NOT EXISTS shard2_server
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'postgres-shard2', port '5432', dbname 'shops_db');

-- Пользовательские маппинги (пароль должен совпадать с указанным в docker-compose.yml)
CREATE USER MAPPING IF NOT EXISTS FOR postgres
    SERVER shard1_server
    OPTIONS (user 'postgres', password 'strongpassword');

CREATE USER MAPPING IF NOT EXISTS FOR postgres
    SERVER shard2_server
    OPTIONS (user 'postgres', password 'strongpassword');

-- Переименование локальной таблицы, чтобы освободить имя shops
ALTER TABLE IF EXISTS shops RENAME TO shops_master;

-- Импорт внешних таблиц из шардов
IMPORT FOREIGN SCHEMA public FROM SERVER shard1_server INTO public;
ALTER FOREIGN TABLE shops RENAME TO shops_shard1;

IMPORT FOREIGN SCHEMA public FROM SERVER shard2_server INTO public;
ALTER FOREIGN TABLE shops RENAME TO shops_shard2;

-- Представление для объединения всех данных
CREATE OR REPLACE VIEW all_shops AS
    SELECT * FROM shops_master
    UNION ALL
    SELECT * FROM shops_shard1
    UNION ALL
    SELECT * FROM shops_shard2;

-- Функция триггера для распределения INSERT
CREATE OR REPLACE FUNCTION shops_insert_trigger()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.region = 'center' THEN
        INSERT INTO shops_shard1 VALUES (NEW.*);
    ELSIF NEW.region = 'siberia' THEN
        INSERT INTO shops_shard2 VALUES (NEW.*);
    ELSE
        RAISE EXCEPTION 'Unknown region: %', NEW.region;
    END IF;
    RETURN NULL; -- отмена вставки в локальную таблицу
END;
$$ LANGUAGE plpgsql;

-- Триггер на таблице shops_master
CREATE TRIGGER shops_insert_trigger
    BEFORE INSERT ON shops_master
    FOR EACH ROW
    EXECUTE FUNCTION shops_insert_trigger();
````

