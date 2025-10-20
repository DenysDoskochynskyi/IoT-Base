# Лабораторна робота №4  
**Предмет:** Основи проєктування IoT  
**Тема:** Розробка бекенду для обробки даних з IoT-системи  

---

### 🔔 Важливе нагадування

Ця лабораторна робота **є продовженням попередніх трьох** і базується на результатах, отриманих у:  
- **Лабораторній роботі №1** — симуляція або створення IoT-пристроїв та надсилання даних;  
- **Лабораторній роботі №2** — збереження даних у базі (AWS DynamoDB або локально через HiveMQ);  
- **Лабораторній роботі №3** — реалізація алгоритму **шифрування та дешифрування MQTT-повідомлень**.  

Отже, у межах цієї лабораторної:  
- дані, які зберігаються у базі даних (**DynamoDB**) або отримуються з брокера (**HiveMQ**),  
  **є зашифрованими відповідно до алгоритму, який ви розробили у ЛР №3**;  
- при виконанні запитів **GET** (`/records`, `/records?date=...`) необхідно **розшифрувати** отримані дані перед їхнім виведенням у відповідь API;  
- при виконанні запиту **POST** (`/notify`) — за потреби можна відправляти як звичайний, так і зашифрований payload.

Приклад розшифрування можна інтегрувати безпосередньо у ваш бекенд — це може бути функція, яка використовує ваш власний алгоритм з попередньої лабораторної (наприклад, Caesar shift, permutation або XOR-підхід).

---

## Мета роботи
- Ознайомитися з принципами побудови бекенд-сервісів для IoT.  
- Реалізувати API, яке може:
  - отримувати дані з бази або джерела телеметрії;
  - фільтрувати дані за датою;
  - надсилати сигнали/команди до IoT Core або HiveMQ-брокера.  
- Навчитися тестувати API за допомогою Postman чи аналогічного середовища.  

---

## Варіант 1: **Cloud реалізація (AWS IoT Core + DynamoDB)**

### Завдання
1. **Розробити простий бекенд (на будь-якій мові, наприклад Python/Node.js/Go)**, який:
   - з’єднується з базою **DynamoDB**;
   - має три основні ендпоінти:
     1. `GET /records` — отримати всі записи з бази;
     2. `GET /records?date=YYYY-MM-DD` — отримати записи за конкретну дату;
     3. `POST /notify` — надіслати повідомлення до конкретного **топіка AWS IoT Core** (наприклад, `iot/lab4/control`).

2. **Налаштувати підключення до DynamoDB**:
   - використовувати AWS SDK (наприклад, `boto3` для Python або `aws-sdk` для Node.js);
   - мати таблицю з полями:  
     `deviceId`, `timestamp`, `value`.

3. **Надсилання повідомлення до IoT Core**:
   - у межах ендпоінта `/notify`, бекенд повинен публікувати повідомлення (наприклад, `"ON"` або `"OFF"`) до вказаного топіка.  
   - це дозволяє симулювати сигнал керування пристроєм (наприклад, вмикання лампи).  

4. **Перевірка роботи через Postman**:
   - Виконати 3 запити:
     - `GET /records` → отримати повний список даних;
     - `GET /records?date=...` → фільтр за датою;
     - `POST /notify` → перевірити, що сигнал дійшов до топіка IoT Core.  

---

### Приклад (Python + Flask + boto3)
```python
from flask import Flask, request, jsonify
import boto3
from datetime import datetime

app = Flask(__name__)

dynamodb = boto3.resource('dynamodb', region_name='eu-north-1')
table = dynamodb.Table('IoT_Lab2_Data')

iot = boto3.client('iot-data', region_name='eu-north-1')

@app.route('/records', methods=['GET'])
def get_records():
    date_filter = request.args.get('date')
    response = table.scan()
    items = response['Items']

    if date_filter:
        items = [x for x in items if x['timestamp'].startswith(date_filter)]
    return jsonify(items)

@app.route('/notify', methods=['POST'])
def send_notification():
    data = request.get_json()
    topic = data.get('topic', 'iot/lab4/control')
    message = data.get('message', 'ON')
    iot.publish(topic=topic, qos=0, payload=message)
    return jsonify({"status": "sent", "topic": topic, "message": message})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

---

## Варіант 2: **Hardware реалізація (HiveMQ + локальне збереження)**

### Завдання
1. **Розробити бекенд (на будь-якій мові)**, який:
   - підключається до **HiveMQ** брокера (`broker.hivemq.com`);
   - підписується на топік, наприклад, `iot/lab4/data`;
   - зберігає отримані повідомлення локально (наприклад, у список, текстовий файл або JSON).  

2. **Реалізувати ті самі 3 ендпоінти:**
   1. `GET /records` — повертає всі отримані MQTT-повідомлення;
   2. `GET /records?date=YYYY-MM-DD` — повертає повідомлення, отримані у конкретний день;
   3. `POST /notify` — надсилає команду (`ON`/`OFF`) до топіка HiveMQ, наприклад, `iot/lab4/cmd`.

3. **Перевірити роботу** через Postman:
   - Виконати послідовно запити `GET` і `POST`;
   - Переконатися, що пристрій або емулятор отримує команди від бекенду.

---

## 🔐 Налаштування доступу та креденшналів

### ✅ Варіант 1: Cloud (AWS IoT Core + DynamoDB)
Для взаємодії з AWS IoT Core та DynamoDB необхідно:

1. **Створити IAM користувача**
   - Увійти до [AWS Management Console](https://aws.amazon.com) → **IAM → Users → Add user**  
   - Обрати *Programmatic access*  
   - Надати політики:  
     - `AmazonDynamoDBFullAccess`  
     - `AWSIoTFullAccess`  
   - Зберегти **Access Key ID** та **Secret Access Key**

2. **Налаштувати AWS CLI**
```bash
pip install awscli
aws configure
```
   Ввести:
```
AWS Access Key ID [None]: YOUR_ACCESS_KEY
AWS Secret Access Key [None]: YOUR_SECRET_KEY
Default region name [None]: eu-north-1
Default output format [None]: json
```

3. **Для Python/Flask**
   AWS SDK (`boto3`) автоматично зчитує креденшнали з CLI або середовища:
```bash
export AWS_ACCESS_KEY_ID=YOUR_ACCESS_KEY
export AWS_SECRET_ACCESS_KEY=YOUR_SECRET_KEY
export AWS_DEFAULT_REGION=eu-north-1
```

4. **Перевірка роботи**
```bash
aws dynamodb list-tables
aws iot list-things
```

---

### ⚙️ Варіант 2: Hardware (HiveMQ + локальне збереження)
1. **Public Broker (без логіну)**  
   Використати безкоштовний брокер:
   ```python
   client.connect("broker.hivemq.com", 1883, 60)
   ```

2. **Власний HiveMQ акаунт**
   Зареєструватися на [HiveMQ Cloud](https://www.hivemq.com/try-out/)  
   Отримати креденшнали:
   ```python
   client.username_pw_set("YOUR_USERNAME", "YOUR_PASSWORD")
   client.connect("YOUR_CUSTOM_BROKER_URL", 8883)
   ```

3. **Перевірити з’єднання**
   - Відкрити [HiveMQ Web Client](https://www.hivemq.com/demos/websocket-client/)
   - Переконатися, що повідомлення надходять у топік `iot/lab4/data` або `iot/lab4/cmd`.

4. **Опціонально: Збереження у файл**
```python
import json
with open("records.json", "w") as f:
    json.dump(records, f, indent=2)
```

---

## Контрольні питання
1. Як бекенд може взаємодіяти з MQTT-брокером?  
2. У чому різниця між AWS IoT Core та HiveMQ?  
3. Як можна організувати збереження даних у базі та локально?  
4. Які формати обміну даними використовуються між бекендом і MQTT?  
5. Як перевірити коректність роботи API через Postman?  
