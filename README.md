# Multiplayer Game Server

Многопользовательский игровой сервер на C++20. Игроки управляют собаками, которые бегают по картам, собирают потерянные предметы и относят их в бюро находок, чтобы заработать очки. Сервер отдаёт REST API, раздаёт статические файлы клиента, сохраняет состояние игры на диск и хранит таблицу рекордов в PostgreSQL.

## Возможности

- Асинхронный HTTP-сервер на Boost.Asio и Boost.Beast. Работает в несколько потоков, а изменения состояния игры идут через `strand`.
- Несколько карт, которые загружаются из JSON-конфигурации. Для каждой карты задаются дороги, здания, бюро находок, вместимость рюкзака и типы предметов.
- Случайная генерация предметов (`loot_generator`) и обнаружение столкновений с предметами и бюро (`collision_detector`).
- Игровой цикл с автоматическим тиком по таймеру или с ручным тиком через API.
- Сохранение и восстановление состояния через Boost.Serialization. Состояние пишется периодически и при остановке сервера.
- Удаление неактивных игроков: игрок, который долго бездействует, уходит из игры, а его результат попадает в таблицу рекордов.
- Таблица рекордов в PostgreSQL (`libpqxx`, пул соединений).
- Структурированное логирование в JSON через Boost.Log.
- Опциональный случайный спавн игроков на дорогах.

## Технологии

C++20, CMake, Conan 1.x, Boost 1.84 (Asio, Beast, JSON, Log, Serialization, Program_options), libpqxx, PostgreSQL, Docker.

## Структура проекта

```
.
├── CMakeLists.txt
├── conanfile.txt
├── Dockerfile
├── data/        # config.json с картами
├── static/      # статические файлы клиента
├── src/         # исходный код сервера
└── tests/
```

Основные модули в `src/`:

| Файл | Назначение |
|---|---|
| `main.cpp` | Разбор аргументов, запуск пула потоков, тикер, сохранение состояния |
| `http_server.*` | Приём соединений и разбор HTTP-запросов |
| `request_handler.*` | Обработка REST API и раздача статики |
| `model.*`, `objects.h`, `geom.h` | Модель игры: карты, дороги, собаки, предметы |
| `loot_generator.*` | Генерация потерянных предметов |
| `collision_detector.*` | Подбор предметов и сдача в бюро находок |
| `state_serialization.h` | Сохранение и восстановление состояния игры |
| `record_repository.*`, `connection_pool.h` | Таблица рекордов в PostgreSQL |
| `json_loader.*`, `extra_data.*` | Загрузка конфигурации карт |

## Сборка

Нужны компилятор с поддержкой C++20, CMake 3.11+, Conan 1.x и PostgreSQL.

```bash
mkdir build && cd build
conan install .. --build=missing -s build_type=Release -s compiler.libcxx=libstdc++11
cmake -DCMAKE_BUILD_TYPE=Release ..
cmake --build .
```

## Запуск

Сервер использует базу данных, поэтому перед запуском нужно задать переменную окружения `GAME_DB_URL`:

```bash
export GAME_DB_URL="postgres://user:password@localhost:5432/game_db"

./game_server \
  --config-file data/config.json \
  --www-root static \
  --tick-period 50 \
  --state-file state.bin \
  --save-state-period 1000 \
  --randomize-spawn-points
```

| Параметр | Описание |
|---|---|
| `-c, --config-file` | Путь к JSON-конфигурации (обязательный) |
| `-w, --www-root` | Каталог со статическими файлами (обязательный) |
| `-t, --tick-period` | Период автоматического тика в миллисекундах. Без него время идёт через `/api/v1/game/tick` |
| `--state-file` | Файл для сохранения и восстановления состояния |
| `--save-state-period` | Период автосохранения в миллисекундах |
| `--randomize-spawn-points` | Появление игроков в случайных точках дорог |
| `-h, --help` | Справка |

Сервер слушает порт `8080`. Он обрабатывает `SIGINT` и `SIGTERM`: при остановке сохраняет состояние и завершается.

## REST API

| Метод | Путь | Описание |
|---|---|---|
| GET, HEAD | `/api/v1/maps` | Список карт |
| GET, HEAD | `/api/v1/maps/{id}` | Описание карты |
| POST | `/api/v1/game/join` | Войти в игру. Тело: `{"userName": "...", "mapId": "..."}`. Ответ: `authToken` и `playerId` |
| GET, HEAD | `/api/v1/game/players` | Список игроков (нужен токен) |
| GET, HEAD | `/api/v1/game/state` | Состояние игроков и предметов (нужен токен) |
| POST | `/api/v1/game/player/action` | Управление движением (нужен токен) |
| POST | `/api/v1/game/tick` | Ручной тик времени. Недоступен, если задан `--tick-period` |
| GET | `/api/v1/game/records` | Таблица рекордов |

Авторизованные запросы передают заголовок `Authorization: Bearer <token>`. Токен состоит из 32 символов и выдаётся при входе в игру.

Пример входа в игру:

```bash
curl -X POST http://localhost:8080/api/v1/game/join \
  -H "Content-Type: application/json" \
  -d '{"userName": "Scooby", "mapId": "map1"}'
```

## Docker

```bash
docker build -t game_server .
docker run --rm -p 8080:8080 -e GAME_DB_URL="postgres://user:password@host:5432/game_db" game_server
```



