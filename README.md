# Лабораторна робота №2 (Апаратний варіант)  
**Предмет:** Основи проєктування IoT  
**Тема:** Робота з MQTT: взаємодія сенсорів та виконавчих пристроїв через Arduino/ESP  

---

## Мета роботи
- Ознайомитися з роботою **MQTT** у поєднанні з фізичними IoT-пристроями.  
- Навчитися відправляти дані з датчиків на MQTT-брокер.  
- Реалізувати керування виконавчим пристроєм (реле, лампа, мотор) через повідомлення з MQTT-брокера.  

---

## Теоретичні відомості
- **MQTT** — легкий протокол обміну повідомленнями у форматі "видавець-підписник".  
- Пристрої можуть не тільки публікувати дані (publish), але й підписуватися на команди (subscribe).  
- Це дозволяє реалізувати **двосторонній зв’язок**:  
  - сенсор → брокер → збереження/обробка даних,  
  - брокер → Arduino → керування виконавчим пристроєм.  

---

## Завдання
1. **Зібрати схему**  
   - Arduino/ESP8266/ESP32.  
   - Один або кілька сенсорів (наприклад: датчик температури, вологості, освітленості).  
   - Виконавчий пристрій (реле, світлодіод, лампа).  

2. **Підключити пристрій до Wi-Fi** та MQTT-брокера (наприклад: `broker.hivemq.com`, порт `1883`).  

3. **Реалізувати відправку даних**  
   - Періодично надсилати показники з датчиків у топік, наприклад:  
     - `iot/lab2/sensor/temp`  
     - `iot/lab2/sensor/humidity`  

4. **Реалізувати підписку на командний топік**  
   - Arduino підписується на топік, наприклад:  
     - `iot/lab2/cmd`  
   - При отриманні повідомлення:  
     - якщо `ON` → увімкнути реле/лампу,  
     - якщо `OFF` → вимкнути реле/лампу.  

5. **Перевірити роботу**  
   - В MQTT-клієнті бачити повідомлення від сенсорів.  
   - Надсилати команду (`ON` / `OFF`) у брокер і перевіряти реакцію виконавчого пристрою.  

---

## Приклад коду (Arduino IDE, ESP8266 + PubSubClient)
```cpp
#include <ESP8266WiFi.h>
#include <PubSubClient.h>

// Налаштування Wi-Fi
const char* ssid = "YOUR_WIFI_SSID";
const char* password = "YOUR_WIFI_PASSWORD";

// MQTT
const char* mqtt_server = "broker.hivemq.com";
WiFiClient espClient;
PubSubClient client(espClient);

const int relayPin = D1; // підключене реле
int sensorValue = 0;     // наприклад, дані з аналогового датчика

void setup_wifi() {
  delay(10);
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
  }
}

void callback(char* topic, byte* payload, unsigned int length) {
  String message;
  for (int i = 0; i < length; i++) {
    message += (char)payload[i];
  }
  
  if (String(topic) == "iot/lab2/cmd") {
    if (message == "ON") {
      digitalWrite(relayPin, HIGH);
    } else if (message == "OFF") {
      digitalWrite(relayPin, LOW);
    }
  }
}

void reconnect() {
  while (!client.connected()) {
    if (client.connect("ESP8266Client")) {
      client.subscribe("iot/lab2/cmd");
    } else {
      delay(5000);
    }
  }
}

void setup() {
  pinMode(relayPin, OUTPUT);
  setup_wifi();
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);
}

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  // Симуляція даних сенсора
  sensorValue = analogRead(A0);
  String payload = String(sensorValue);
  client.publish("iot/lab2/sensor/temp", payload.c_str());

  delay(3000);
}
```

---

## Очікуваний результат
- В MQTT-клієнті відображаються дані з сенсора.  
- Виконавчий пристрій (реле/лампа) реагує на команду, надіслану у брокер (`ON` / `OFF`).  
- Робота підтверджена скріншотами з MQTT-клієнта та фото/відео з Arduino.  

---

## Додаткові завдання (за бажанням)
- Використати кілька сенсорів і передавати дані у різні топіки.  
- Реалізувати логіку: якщо температура перевищує певне значення → автоматично надсилати команду в інший топік.  
- Додати OLED-дисплей для відображення стану сенсорів і команд.  

---

## Звіт
У звіті повинно бути:  
1. Схема підключення Arduino/ESP.  
2. Код програми.  
3. Скріншоти з MQTT-клієнта (дані від сенсорів, команди).  
4. Фото або скріншоти роботи виконавчих пристроїв.  
5. Висновки.  

---

## Контрольні питання
1. Як працює протокол MQTT (publish/subscribe)?  
2. Як організувати двосторонню взаємодію через брокер?  
3. Чим відрізняється симуляція від роботи з реальним сенсором?  
4. Як перевірити коректність виконання команд на Arduino?  
5. Які ще виконавчі пристрої можна підключити через MQTT?  

---
