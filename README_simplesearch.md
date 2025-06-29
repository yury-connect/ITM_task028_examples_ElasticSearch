

Ссылка на [оригинальный репозиторий](https://github.com/SuchkovDenis/elasticexample)

---
- **PostgreSQL** будет хранить основные данные приложения _(фильмы, пользователей и т. д.)_.  
- **Elasticsearch** обеспечит быстрый полнотекстовый поиск по данным из PostgreSQL.  

---
### Запустить `PostgreSQL` в `Docker`:
```bash
docker run --name postgres_db -e POSTGRES_DB=movies -e POSTGRES_USER=user -e POSTGRES_PASSWORD=1234 -p 5432:5432 -d postgres:16.0
```

> #### Параметры команды:
> - `--name postgres_db`  
> Задает имя контейнеру (`postgres_db`), чтобы можно было легко ссылаться на него через `docker logs`, `docker stop` и т. д.  
>
> - `-e POSTGRES_DB=movies`  
> Устанавливает переменную окружения для создания базы данных `movies` при старте контейнера.  
>
> - `-e POSTGRES_USER=user`  
> Создает пользователя `user` с правами доступа к БД.  
>
> - `-e POSTGRES_PASSWORD=1234`  
> Устанавливает пароль `1234` для пользователя `user`.  
>
> - `-p 5432:5432`  
> Пробрасывает порт `5432` из контейнера на порт `5432` хоста.  
> Формат: `-p <порт_хоста>:<порт_контейнера>`  
>
> - `-d`  
> Запускает контейнер в **фоновом** режиме _(detached mode)_.  
>
> - `postgres:16.0`  
> Использует **официальный** образ PostgreSQL версии 16.0.  
>

### Запустить `Elasticsearch` в `Docker`:
```bash
docker run -p 9200:9200 -e "discovery.type=single-node" -d docker.elastic.co/elasticsearch/elasticsearch:7.17.22
```
> #### Параметры команды:
> - `-p 9200:9200`  
    Пробрасывает порт `9200` (_HTTP-API Elasticsearch_) из контейнера на порт `9200` хоста.  
> 
> - `-e "discovery.type=single-node"`  
> Указывает Elasticsearch работать в режиме **одного узла** (_без кластера_).  
> Это необходимо для **локальной** разработки, чтобы Elasticsearch не пытался искать другие узлы.    
> 
> - `-d`   
> Запускает контейнер в фоновом режиме.  
> 
> - `docker.elastic.co/elasticsearch/elasticsearch:7.17.22`  
> Использует официальный образ Elasticsearch версии `7.17.22` (_совместимый с Spring Boot 3.3.2_).  
> 

---
## Проверка работы контейнеров
### 1. Проверить статус контейнеров:
```bash
docker ps
```
Должны быть видны оба контейнера (`postgres_db` и Elasticsearch).

### 2. Проверить логи PostgreSQL:
```bash
docker logs postgres_db
```
Должны быть видны сообщения об успешном запуске PostgreSQL `database system is ready to accept connections`.

### 3. Проверить логи Elasticsearch:
```bash
docker logs <container_id_elasticsearch> 
```
Значение `<container_id_elasticsearch>` можно получить командой:
```bash
docker ps
```

В логах должно быть `started` и статус `GREEN`.

### 4. Проверить подключение к PostgreSQL: 
```bash
psql -h localhost -U user -d movies
```
Предварительно утилита `psql` _(клиент PostgreSQL)_ должны быть установлена на вашей системе.

### 5. Проверить Elasticsearch API:
```bash
curl http://localhost:9200
```
Должен вернуться JSON с информацией о версии Elasticsearch.

---
## Если используете Docker для проверки PostgreSQL
#### 1. Зайдите в контейнер с PostgreSQL:
```bash
docker exec -it postgres_db bash
```

#### 2. Подключитесь к БД через psql внутри контейнера:
```sql
psql -U user -d movies
```
(Пароль: `1234`)

#### 3. Выполните SQL-запрос:
```sql
select * from movies
```
или
```sql
SELECT version();
```
по завершении:
```sql
\q  -- для выхода
```
и
```sql
exit  -- закроеет подключение к контейнеру
```

---
Настройки подключения в DBeaver:
> Host: `localhost`
> Port: `5432`
> Database: `movies`
> Username: `user`
> Password: `1234`

---

## ТРОЙНАЯ КОМАНДА: <br/> - Удаляет папку target/ <br/> - Компилирует исходный код / <br/> - Собирает Docker-образ

```bash
mvn clean compile jib:dockerBuild
```

> 1. `mvn clean` 
> Удаляет папку `target/` (если она существует).
> 
> 2. `mvn compile`
> Компилирует исходный код (`src/main/java`) в байт-код (`.class`-файлы) и помещает их в `target/classes`.
> Копирует ресурсы (src/main/resources) в target/classes (Чтобы подготовить артефакты для упаковки в Docker-образ).
> 
> 3. `mvn jib:dockerBuild`
> Собирает Docker-образ вашего приложения без необходимости установки Docker или написания Dockerfile.
> Использует плагин Jib от Google.
> **Шаги**, которые выполняет **Jib**:
> > - Скачивает базовый образ (`eclipse-temurin:21-jre` из вашего `pom.xml`).
> > - Копирует:
> > > - Скомпилированные классы (`target/classes`),
> > > - Зависимости (JAR-файлы из `target/lib`),
> > > - Ресурсы (`src/main/resources`).
> > - Создает слои образа, оптимизируя кэширование.
> > - Загружает собранный образ в **локальный Docker Daemon** (название образа — `moviesearch`, как указано в `pom.xml`).
 
> **Зачем использовать Jib вместо Dockerfile:**
> - Не требует установки Docker на этапе сборки (хотя для `dockerBuild` он нужен).
> - Создает более эффективные образы (мультислойные, с кэшированием зависимостей).
> - Безопасность: не требует root-доступа, как `docker build`. 
> 

### Аналог с Dockerfile
Если бы вы не использовали Jib, команда выглядела бы так:
```bash
mvn clean package  # Собирает JAR-файл
docker build -t moviesearch .  # Собирает образ из Dockerfile
```
Но Jib делает это автоматически, без лишних шагов.

### Как проверить, что образ создался
После успешного выполнения `jib:dockerBuild`:
```bash
docker images
```
Вы должны увидеть образ moviesearch в списке. Затем его можно запустить:
```bash
docker run -p 8080:8080 moviesearch
```

---
## Запускаем только два сервиса:
```bash
docker-compose up app db   # Запускаем только два сервиса
```

1. **Запуск только указанных сервисов**  
Docker Compose ищет в файле `docker-compose.yml` сервисы с именами `app` и `db` и запускает их 
в указанном порядке (с учётом зависимостей `depends_on`).  
Elasticsearch (`elasticsearch`) **не будет запущен**, так как он не указан в команде.  


2. **Порядок запуска**  
- Сначала стартует `db` (PostgreSQL), так как сервис `app` зависит от него (`depends_on: db`).  
- Затем запускается `app` (ваше Spring Boot-приложение).  

3. **Что делает каждый сервис?**  
- db:  
  - Создаёт контейнер из образа postgres:16.0.  
  - Настраивает БД movies с пользователем movies_user и паролем movies_password.  
  - Открывает порт 5432 на хосте.  

- app:  
  - Собирает образ из Dockerfile (если он ещё не собран) или использует существующий moviesearch.  
  - Подключается к PostgreSQL по адресу db:5432 (используя переменные окружения из environment).  
  - Открывает порт 8080 на хосте.  

### Как проверить, что всё работает?
1. Логи контейнеров
   Отдельно просмотрите логи каждого сервиса:
```bash
docker-compose logs app  # Логи приложения
docker-compose logs db   # Логи PostgreSQL
```
2. Проверка PostgreSQL
   Подключитесь к БД через psql (если установлен):
```bash
psql -h localhost -U movies_user -d movies
```
Пароль: `1234`.

3. Проверка приложения
   Подключитесь к приложению через браузер по адресу http://localhost:8080.
```bash
http://localhost:8080
```
Или проверьте API:
```bash
curl http://localhost:8080
```

---
## Дополнительные команды

Запуск в фоновом режиме:
```bash
docker-compose up -d app db
```

Остановка сервисов:
```bash
docker-compose stop app db
```
Удаление контейнеров:
```bash
docker-compose down
```

---
