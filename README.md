# orders-api

Микросервис на Go, который получает данные заказов из Kafka, сохраняет их в PostgreSQL и кеширует в памяти для быстрого отдачи через HTTP‑API и простую веб‑страницу.

## Что делает сервис
- Подписывается на тему Kafka и обрабатывает JSON‑сообщения с моделью заказа.
- Валидирует и сохраняет заказы в PostgreSQL в рамках транзакций.
- Кеширует последние заказы в памяти (sync.Map) для ускорения повторных запросов.
- Восстанавливает кеш из базы при старте.
- Возвращает заказ по `order_uid` через JSON‑API и простую HTML‑страницу.

[Смотреть демонстрацию](docs/video/demonstration.mp4)

## Быстрый старт и проверка
1. Скопируйте шаблоны конфигураций:
```bash
cp deploy/.env.example deploy/.env
cp configs/.config.yml.example configs/config.yml
```

2. Перейдите в каталог `deploy` и соберите/запустите Docker Compose:
```bash
cd deploy
docker compose build --no-cache
docker compose up -d
```

3. После запуска проверьте, что контейнеры работают:
```bash
docker ps
```

4. HTTP‑проверки:
```bash
curl -s http://host:port/health
# curl -s http://localhost:8080/health
curl -s http://host:port/static/index.html
# curl -s http://localhost:8080/static/index.html
```

5. Отправьте тестовое сообщение в Kafka (пример — внутри контейнера Kafka):
```bash
# host = localhost
# port = 9092
docker exec -it kafka sh -c 'echo "{\"order_uid\": \"b563feb7b2b84b6test\", \"track_number\": \"WBILMTESTTRACK\", \"entry\": \"WBIL\", \"delivery\": {\"name\": \"Test Testov\", \"phone\": \"+9720000000\", \"zip\": \"2639809\", \"city\": \"Kiryat Mozkin\", \"address\": \"Ploshad Mira 15\", \"region\": \"Kraiot\", \"email\": \"test@gmail.com\"}, \"payment\": {\"transaction\": \"b563feb7b2b84b6test\", \"request_id\": \"\", \"currency\": \"USD\", \"provider\": \"wbpay\", \"amount\": 1817, \"payment_dt\": 1637907727, \"bank\": \"alpha\", \"delivery_cost\": 1500, \"goods_total\": 317, \"custom_fee\": 0}, \"items\": [{\"chrt_id\": 9934930, \"track_number\": \"WBILMTESTTRACK\", \"price\": 453, \"rid\": \"ab4219087a764ae0btest\", \"name\": \"Mascaras\", \"sale\": 30, \"size\": \"0\", \"total_price\": 317, \"nm_id\": 2389212, \"brand\": \"Vivienne Sabo\", \"status\": 202}], \"locale\": \"en\", \"internal_signature\": \"\", \"customer_id\": \"test\", \"delivery_service\": \"meest\", \"shardkey\": \"9\", \"sm_id\": 99, \"date_created\": \"2021-11-26T06:22:19Z\", \"oof_shard\": \"1\"}" | kafka-console-producer --broker-list host:port --topic orders'
```

6. Посмотрите логи сервера:
```bash
docker compose logs server
```

7. Проверьте кеш / API:
```bash
curl -s http://host:port/order/b563feb7b2b84b6test | jq . 2>/dev/null || curl -s http://host:port/order/b563feb7b2b84b6test
# curl -s http://localhost:8080/order/b563feb7b2b84b6test | jq . 2>/dev/null || curl -s http://localhost:8080/order/b563feb7b2b84b6test
```

8. Проверьте данные в PostgreSQL (пример):
```bash
docker exec -it postgres psql -U postgres -d postgres -c "SELECT * FROM orders WHERE order_uid = 'b563feb7b2b84b6test';"
docker exec -it postgres psql -U postgres -d postgres -c "SELECT * FROM delivery WHERE order_uid = 'b563feb7b2b84b6test';"
docker exec -it postgres psql -U postgres -d postgres -c "SELECT * FROM payment WHERE order_uid = 'b563feb7b2b84b6test';"
docker exec -it postgres psql -U postgres -d postgres -c "SELECT * FROM items WHERE order_uid = 'b563feb7b2b84b6test';"
```

9. Перезапустите сервер и убедитесь, что кеш восстановился:
```bash
docker compose restart server
docker compose logs server
```

10. Остановите сервисы:
```bash
docker compose stop
```

## Архитектура и расположение кода (структура пакетов)
```
- cmd/
  - service/             # точка входа
- configs/               # YAML‑файлы конфигураций 
- deploy/                # docker-compose, примеры .env
- docs/
  - db/                  # документация по базе
  - http/                # HTTP‑эндпоинты (endpoints.http)
  - scheme/              # схема
  - video/               # ссылки на демонстрации
- internal/
  - config/              # загрузка и структура конфигурации (Viper)
  - di/                  # инъекция зависимостей и сборка компонентов
  - domain/              # доменные модели (структуры)
  - ports/               # интерфейсы (порты приложения)
  - app/
    - order/             # чистая бизнес‑логика и валидация
  - adapters/
    - cache/             # in‑memory кеш
    - db/
      - postgres/        # соединение, репозиторий, миграции
        - connect/
        - migration/
    - kafka/             # адаптер потребителя Kafka (segmentio/kafka-go)
    - server/            # HTTP‑сервер (Fiber v3) и обработчики
  - logger/              # обёртка для zap
  - shutdown/            # помощник для корректного завершения
- migrations/            # SQL‑миграции
- static/                # статическая страница (index.html)
```
## Эндпоинты
Доступные эндпоинты: [docs/http/endpoints.http](docs/http/endpoints.http)
- GET /health — проверка состояния
- GET /static/index.html — фронтенд
- GET /order/{order_uid} — получить заказ по UID (JSON)

## Используемые библиотеки
- github.com/gofiber/fiber/v3 
- github.com/golang-migrate/migrate/v4 
- github.com/jackc/pgx/v5 
- github.com/segmentio/kafka-go
- github.com/spf13/viper 
- go.uber.org/zap 
- golang.org/x/sync 


