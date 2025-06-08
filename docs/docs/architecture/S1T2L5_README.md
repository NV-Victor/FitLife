- [Введение](#введение)
- [Базовая инструкция](#базовая-инструкция)
  - [Микросервис Order Service Node](#микросервис-order-service-node)
    - [Подготовка приложения order\_service](#подготовка-приложения-order_service)
    - [Подготовка Docker для order\_service](#подготовка-docker-для-order_service)
  - [Микросервис Logistics Service Node](#микросервис-logistics-service-node)
    - [Подготовка приложения logistics-service](#подготовка-приложения-logistics-service)
    - [Подготовка Docker для logistics-service](#подготовка-docker-для-logistics-service)
  - [Подготовка Docker Compose](#подготовка-docker-compose)
  - [Работа с Docker Compose](#работа-с-docker-compose)
  - [Проверка работоспособности сервисов](#проверка-работоспособности-сервисов)
    - [Проверка сервиса POST /orders](#проверка-сервиса-post-orders)
    - [Проверка сервиса PUT /orders/1](#проверка-сервиса-put-orders1)
    - [Проверка сервиса POST /deliveries](#проверка-сервиса-post-deliveries)
    - [Проверка сервиса PUT /deliveries/1](#проверка-сервиса-put-deliveries1)
    - [Проверка базы данных orders](#проверка-базы-данных-orders)
    - [Проверка базы данных deliveries](#проверка-базы-данных-deliveries)
- [Полезные команды](#полезные-команды)

# Введение

Делаем настройки из урока: Ссылка на урок: [Спринт 1/11: Спринт 1. 26.05 - 08.06 МД 🟢 → Тема 2/4: Микросервисы и документирование решений → Урок 5/6: Расширенная контейнеризация и Docker Compose]( https://practicum.yandex.ru/learn/software-architect/courses/958ef1df-e5b3-4055-b01e-540e479ca32f/sprints/573874/topics/9f086841-5d67-43d6-981e-b5d5af07d6de/lessons/a6abafe8-d9c8-44ce-9180-1ade7f58af72/#bc6441a6-b794-4fe3-8139-4a44bdfc56dd )


Предполагаем, что у нас есть два микросервиса: `Order Service Node` и `Logistics Service Node`.

**Order Service Node**:
- имеет сервис `POST /orders`, который создает в БД order с информацией Имя клиента и Товар. Возвращается 201 Order created successfully. Order создается со статусом *Pending*.
- имеет сервис `PUT /orders/{order_id}`, который обновляет статус order по его id.  Возвращает 201 Order updated successfully, если заказ найден. Или 404 Order not found, если заказ не найден.
- структура репозитория:

```
order-service/
├── app.py
├── requirements.txt
├── Dockerfile
├── swagger.yml
├── db/
│   └── models.py
└── utils/
    └── db_utils.py
```

**Logistics Service Node**
- имеет сервис `POST /deliveries`, который создает запись delivery с order_id и в статусе In Transit. Возвращает 201 Delivery created successfully.
- имеет сервис `PUT /deliveries`, который обновляет статус delivery по ее id.  Возвращает 201 Delivery updated successfully, если запись доставки  найдена. Или 404 Delivery not found, если запись доставки не найдна.
- структура репозитория:

```
logistics-service/
├── app.py
├── requirements.txt
├── Dockerfile
├── swagger.yml
├── db/
│   └── models.py
└── utils/
    └── db_utils.py
```


# Базовая инструкция
## Микросервис Order Service Node
### Подготовка приложения order_service

1. Создать папку `order-service`.
   
*Папку можно положить в любое место, главное потом в нее перейте. Я создал здесь `/home/ubuntu/YandexPracticum/ArchPO/Practice/S1T2L5/order_service/`*

2. Положить в папку файл приложения `app.py` со следующим содержимым:

```Python
from flask import Flask, request, jsonify
from db.models import db, Order
from utils.db_utils import init_db
import os

app = Flask(__name__)

#app.config['SQLALCHEMY_DATABASE_URI'] = 'postgresql://postgres:mysecretpassword@db:5432/orders'

app.config['SQLALCHEMY_DATABASE_URI'] = os.getenv("DATABASE_URL")

db.init_app(app)

@app.route('/orders', methods=['POST'])
def create_order():
    data = request.get_json()
    new_order = Order(customer_name=data['customer_name'], item=data['item'], status='Pending')
    db.session.add(new_order)
    db.session.commit()
    return jsonify({'message': 'Order created successfully'}), 201

@app.route('/orders/<int:order_id>', methods=['PUT'])
def update_order(order_id):
    data = request.get_json()
    order = Order.query.get(order_id)
    if not order:
        return jsonify({'message': 'Order not found'}), 404
    order.status = data['status']
    db.session.commit()
    return jsonify({'message': 'Order updated successfully'})

if __name__ == '__main__':
    with app.app_context():
        init_db()
    app.run(host='0.0.0.0', port=5000)
```
3. Положить в папку файл зависимостей `requirements.txt` со следующим содержимым:

```
flask
flask_sqlalchemy
psycopg2-binary
```
4. Положить в папку файл сваггера `swagger.yml` со следующим содержимым:

```yaml
swagger: '2.0'
info:
  title: Order Service API
  version: 1.0.0
paths:
  /orders:
    post:
      summary: Create a new order
      consumes:
        - application/json
      parameters:
        - in: body
          name: body
          required: true
          schema:
            type: object
            properties:
              customer_name:
                type: string
              item:
                type: string
      responses:
        201:
          description: Order created successfully
  /orders/{order_id}:
    put:
      summary: Update an order
      consumes:
        - application/json
      parameters:
        - in: path
          name: order_id
          required: true
          type: integer
        - in: body
          name: body
          required: true
          schema:
            type: object
            properties:
              status:
                type: string
      responses:
        200:
          description: Order updated successfully
        404:
          description: Order not found
```

5. Создать папку `order_service/db/`.
6. Положить в нее файл `models.py` со следующим содержимым:

```python
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()

class Order(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    customer_name = db.Column(db.String(80), nullable=False)
    item = db.Column(db.String(120), nullable=False)
    status = db.Column(db.String(20), nullable=False)
```

7. Создать папку `order_service/utils/`
8. Положить в нее файл `db_utils.py` со следующим содержимым:
```python
from db.models import db

def init_db():
    db.create_all()
```

### Подготовка Docker для order_service

1. Положить файл `DockerFile` в папку `order_service` со следующим содержимым:

```dockerfile
# Используем базовый образ Python
FROM python:3.8-slim

# Устанавливаем рабочую директорию
WORKDIR /app

# Копируем файлы приложения в контейнер
COPY . /app

# Устанавливаем зависимости
RUN pip install --no-cache-dir -r requirements.txt

# Определяем команду для запуска приложения
CMD ["python", "app.py"]
```

## Микросервис Logistics Service Node
### Подготовка приложения logistics-service

1. Создать папку `logistics-service`.
   
*Папку можно положить в любое место, главное потом в нее перейте. Я создал здесь `/home/ubuntu/YandexPracticum/ArchPO/Practice/S1T2L5/logistics-service/`*

2. Положить в папку файл приложения `app.py` со следующим содержимым:

```Python
from flask import Flask, request, jsonify
from db.models import db, Delivery
from utils.db_utils import init_db
import os

app = Flask(__name__)

# app.config['SQLALCHEMY_DATABASE_URI'] = 'postgresql://postgres:password@db:5432/logistics'

app.config['SQLALCHEMY_DATABASE_URI'] = os.getenv("DATABASE_URL")

db.init_app(app)

@app.route('/deliveries', methods=['POST'])
def create_delivery():
    data = request.get_json()
    new_delivery = Delivery(order_id=data['order_id'], status='In Transit')
    db.session.add(new_delivery)
    db.session.commit()
    return jsonify({'message': 'Delivery created successfully'}), 201

@app.route('/deliveries/<int:delivery_id>', methods=['PUT'])
def update_delivery(delivery_id):
    data = request.get_json()
    delivery = Delivery.query.get(delivery_id)
    if not delivery:
        return jsonify({'message': 'Delivery not found'}), 404
    delivery.status = data['status']
    db.session.commit()
    return jsonify({'message': 'Delivery updated successfully'})

if __name__ == '__main__':
    with app.app_context():
        init_db()
    app.run(host='0.0.0.0', port=5001)
```
3. Положить в папку файл зависимостей `requirements.txt` со следующим содержимым:

```
flask
flask_sqlalchemy
psycopg2-binary
```
4. Положить в папку файл сваггера `swagger.yml` со следующим содержимым:

```yaml
swagger: '2.0'
info:
  title: Logistics Service API
  version: 1.0.0
paths:
  /deliveries:
    post:
      summary: Create a new delivery
      consumes:
        - application/json
      parameters:
        - in: body
          name: body
          required: true
          schema:
            type: object
            properties:
              order_id:
                type: integer
      responses:
        201:
          description: Delivery created successfully
  /deliveries/{delivery_id}:
    put:
      summary: Update a delivery
      consumes:
        - application/json
      parameters:
        - in: path
          name: delivery_id
          required: true
          type: integer
        - in: body
          name: body
          required: true
          schema:
            type: object
            properties:
              status:
                type: string
      responses:
        200:
          description: Delivery updated successfully
        404:
          description: Delivery not found
```

5. Создать папку `logistics-service/db/`.
6. Положить в нее файл `models.py` со следующим содержимым:

```python
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()

class Delivery(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    order_id = db.Column(db.Integer, nullable=False)
    status = db.Column(db.String(20), nullable=False)
```

7. Создать папку `logistics-service/utils/`
8. Положить в нее файл `db_utils.py` со следующим содержимым:
```python
from db.models import db

def init_db():
    db.create_all()
```

### Подготовка Docker для logistics-service

1. Положить файл `DockerFile` в папку `logistics-service` со следующим содержимым:

```dockerfile
# Используем базовый образ Python
FROM python:3.8-slim

# Устанавливаем рабочую директорию
WORKDIR /app

# Копируем файлы приложения в контейнер
COPY . /app

# Устанавливаем зависимости
RUN pip install --no-cache-dir -r requirements.txt

# Определяем команду для запуска приложения
CMD ["python", "app.py"]
```

## Подготовка Docker Compose

1. Положить рядом с папками `order_service` и `logistics-service` файл `docker-compose.yml` со следующим содержимым:

*У меня получился путь: `/home/ubuntu/YandexPracticum/ArchPO/Practice/S1T2L5/`*

```yaml
version: '3.8'

services:
  order-service:
    build: ./order-service
    container_name: order-service-v3
    #command: tail -f /dev/null    #если нужно контейнер оставить рабочим в случае ошибки выполнения app.py
    ports:
      - "5000:5000"
    environment:
      - DATABASE_URL=postgresql://postgres:${POSTGRES_PASSWORD}@db:5432/orders
    depends_on:
      - db
    networks:
      - quickdelivery-network

  logistics-service:
    build: ./logistics-service
    container_name: logistics-service
    #command: tail -f /dev/null    #если нужно контейнер оставить рабочим в случае ошибки выполнения app.py
    ports:
      - "5001:5001"
    environment:
      - DATABASE_URL=postgresql://postgres:${POSTGRES_PASSWORD}@db:5432/logistics
    depends_on:
      - db
    networks:
      - quickdelivery-network

  db:
    image: postgres:13
    container_name: postgres
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
      - postgres-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    networks:
      - quickdelivery-network

networks:
  quickdelivery-network:
    driver: bridge

volumes:
  postgres-data:
```
2. Положить рядом с папками `order_service` и `logistics-service` файл `init.sql` со следующим содержимым:

```sql
CREATE DATABASE orders;
CREATE DATABASE logistics;
```

3. Положить рядом с папками `order_service` и `logistics-service` файл `.env` со следующим содержимым:

```
POSTGRES_PASSWORD=mysecretpassword
```

## Работа с Docker Compose

1. Запускаем сервисы командой:

Предварительно переходим в директорию с файлом `docker-compose.yml`

```bash
docker-compose up --build 
```

## Проверка работоспособности сервисов

### Проверка сервиса POST /orders

1. Выполнить команду для создания заказа:

```bash
curl -X POST -H "Content-Type: application/json" -d '{"customer_name": "John Doe", "item": "Laptop"}' http://localhost:5000/orders 
```

*В результате будет сообщение: `{"message":"Order created successfully"}`*

### Проверка сервиса PUT /orders/1

1. Выполнить команду для обновления существующего заказа:

```bash
curl -X PUT -H "Content-Type: application/json" -d '{"status": "Shipped"}' http://localhost:5000/orders/1
```

*В результате будет сообщение: `{"message":"Order updated successfully"}`*

2. Выполнить команду для обновления несуществующего заказа:

```bash
curl -X PUT -H "Content-Type: application/json" -d '{"status": "Shipped"}' http://localhost:5000/orders/123
```

*В результате будет сообщение: `{"message":"Order not found"}`*

### Проверка сервиса POST /deliveries

1. Выполнить команду для создания доставки:

```bash
curl -X POST -H "Content-Type: application/json" -d '{"order_id": 1}' http://localhost:5001/deliveries 
```

*В результате будет сообщение: `{"message":"Delivery created successfully"}`*

### Проверка сервиса PUT /deliveries/1

1. Выполнить команду для обновления существующей доставки:

```bash
curl -X PUT -H "Content-Type: application/json" -d '{"status": "Delivered"}' http://localhost:5001/deliveries/1
```

*В результате будет сообщение: `{"message":"Delivery updated successfully"}`*

2. Выполнить команду для обновления несуществующей доставки:

```bash
curl -X PUT -H "Content-Type: application/json" -d '{"status": "Delivered"}' http://localhost:5001/deliveries/123
```

*В результате будет сообщение: `{"message":"Delivery not found"}`*

### Проверка базы данных orders

1. Выполнить команду для подключения к БД orders в контейнере

```bash
docker exec -it postgres psql -U postgres -d orders
```
2. Выполнить селект:
```sql
SELECT * FROM "order";
```
3. Выйти из контейнера командой `exit`.

### Проверка базы данных deliveries

1. Выполнить команду для подключения к БД deliveries в контейнере

```bash
docker exec -it postgres psql -U postgres -d logistics
```
2. Выполнить селект:
```sql
SELECT * FROM "delivery";
```
3. Выйти из контейнера командой `exit`.

# Полезные команды

- **docker network ls**                                        - просмотр всех сетей
- **docker network inspect s1t2l5_quickdelivery-network**      - просмотр состояния сети
- **docker-compose up --build**                                - сборка через Docker Compose
- **docker-compose down --rmi all**                            - удалить все образы через Docker Compose
- **docker-compose down -v**                                   - удалить все контейнеры и тома через Docker Compose
- **docker exec -it order-service-v3 /bin/bash**               - открыть терминал в контейнее
- **docker exec -it order-service-v3 env**                     - посмотреть переменные окружения внутри контейнера
- **docker exec -it postgres psql -U postgres**                - подключиться к postgre в контейнере