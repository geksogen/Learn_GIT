# MQTT_IoT

## Архитектура
Архитектура системы включает следующие компоненты:

1. **Мобильное приложение**: Отправляет команды на включение и выключение лампочки и получает текущее состояние лампочки.
2. **IoT-лампочка**: Получает команды и публикует текущее состояние.
3. **MQTT-брокер**: Управляет обменом сообщениями между мобильным приложением и IoT-лампочкой.

``` plantuml
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4.puml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Context.puml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

LAYOUT_LANDSCAPE()
    title Архитектура взаимодействия с IoT-лампочкой через MQTT брокер

    Person(user, "Пользователь", "Отправляет команды на включение и выключение лампочки.")

    System_Boundary(system, "Система управления IoT-лампочкой") {
        Container(mobile_app, "Мобильное приложение", "Python, MQTT", "Отправляет команды и получает состояние лампочки.")
        Container(mqtt_broker, "MQTT-брокер", "MQTT", "Управляет обменом сообщениями.")
        Container(ecu, "IoT-лампочка (ECU)", "Python, MQTT", "Получает команды и публикует состояние.")
    }

    Rel(user, mobile_app, "Отправляет команды")
    Rel(mobile_app, mqtt_broker, "Публикует команды и подписывается на состояние")
    Rel(ecu, mqtt_broker, "Получает команды и публикует состояние")
@enduml
```

## Функциональные требования

 **FR_01 Управление лампочкой**: Мобильное приложение должно иметь возможность отправлять команды на включение и выключение IoT.
 **FR_02 Получение состояния**: Мобильное приложение должно получать текущее состояние IoT (включена/выключена).
 **FR_03 Обработка ошибок**: В случае, если команда не выполнена, мобильное приложение должно отображать сообщение об ошибке "Команда не выполнена".

## Логика работы

 **Отправка команды**:
   - Пользователь отправляет команду (ON/OFF) через мобильное приложение.
   - Мобильное приложение публикует команду в топик `command`.

 **Ожидание ответа**:
   - После отправки команды запускается таймер.
   - Если в течение заданного времени не поступает обновление состояния, выводится сообщение об ошибке.

 **Получение состояния**:
   - IoT получает команду из топика `command` и выполняет соответствующее действие.
   - После выполнения команды IoT публикует свой текущий статус в топик `state`.
   - Мобильное приложение получает обновление состояния из топика `state` и обновляет интерфейс пользователя.

## Sequence Diagram

``` mermaid
sequenceDiagram
    participant User
    participant MobileApp
    participant IoT
    participant CommandTopic as Топик "command"
    participant StateTopic as Топик "state"

    User->>MobileApp: Отправить команду (ON/OFF)
    MobileApp->>CommandTopic: Публикация команды
    Note right of MobileApp: Запуск таймера

    alt Команда выполнена
        CommandTopic-->>IoT: Получение команды
        IoT->>StateTopic: Публикация состояния (ON)
        StateTopic-->>MobileApp: Получение состояния (ON)
        MobileApp->>User: Обновление интерфейса (Лампочка: ON)
        MobileApp->>MobileApp: Сброс таймера
    else Команда не выполнена
        Note right of MobileApp: Таймер истек
        MobileApp->>User: Отображение ошибки "Команда не выполнена"
    end
```

## Пример реализации концепта на Python

### Мобильное приложение

```python
import paho.mqtt.client as mqtt
import threading

# Настройки MQTT
BROKER_ADDRESS = "mqtt.example.com"
COMMAND_TOPIC = "command"
STATE_TOPIC = "state"
COMMAND_TIMEOUT = 5  # Таймаут в секундах

# Флаг для отслеживания получения состояния
state_received = False

# Функция обратного вызова при подключении
def on_connect(client, userdata, flags, rc):
    print("Connected with result code " + str(rc))
    client.subscribe(STATE_TOPIC)

# Функция обратного вызова при получении сообщения
def on_message(client, userdata, msg):
    global state_received
    if msg.topic == STATE_TOPIC:
        state_received = True
        print(f"Лампочка: {msg.payload.decode()}")
        # Обновите интерфейс пользователя

# Функция для отправки команды
def send_command(command):
    global state_received
    state_received = False
    client.publish(COMMAND_TOPIC, command)
    # Запуск таймера
    timer = threading.Timer(COMMAND_TIMEOUT, check_command_status)
    timer.start()

# Функция для проверки статуса команды
def check_command_status():
    if not state_received:
        print("Ошибка: Команда не выполнена")
        # Отобразите сообщение об ошибке в интерфейсе пользователя

# Создание клиента MQTT
client = mqtt.Client()
client.on_connect = on_connect
client.on_message = on_message

# Подключение к брокеру
client.connect(BROKER_ADDRESS, 1883, 60)

# Пример использования
send_command("ON")  # Включение лампочки
send_command("OFF")  # Выключение лампочки

# Запуск цикла обработки сообщений
client.loop_forever()
```
## Пример реализации концепта на Python
### IoT лампочка
```python
import paho.mqtt.client as mqtt

# Настройки MQTT
BROKER_ADDRESS = "mqtt.example.com"
COMMAND_TOPIC = "command"
STATE_TOPIC = "state"

# Функция обратного вызова при подключении
def on_connect(client, userdata, flags, rc):
    print("Connected with result code " + str(rc))
    client.subscribe(COMMAND_TOPIC)

# Функция обратного вызова при получении сообщения
def on_message(client, userdata, msg):
    if msg.topic == COMMAND_TOPIC:
        command = msg.payload.decode()
        if command == "ON":
            turn_on()
        elif command == "OFF":
            turn_off()

# Функции для управления лампочкой
def turn_on():
    print("Лампочка включена")
    client.publish(STATE_TOPIC, "ON")

def turn_off():
    print("Лампочка выключена")
    client.publish(STATE_TOPIC, "OFF")

# Создание клиента MQTT
client = mqtt.Client()
client.on_connect = on_connect
client.on_message = on_message

# Подключение к брокеру
client.connect(BROKER_ADDRESS, 1883, 60)

# Запуск цикла обработки сообщений
client.loop_forever()
```
## Преимущества MQTT перед REST API для взаимодействия с IoT-устройствами

Взаимодействие с IoT-устройствами через MQTT имеет несколько преимуществ по сравнению с использованием REST API. Вот основные причины, по которым MQTT часто предпочитают для IoT-приложений:

 **Эффективность и экономия ресурсов**:
   - **MQTT** использует легковесный протокол, который минимизирует объем передаваемых данных. Это особенно важно для IoT-устройств, которые часто имеют ограниченные вычислительные ресурсы и работают на низкоскоростных сетях.
   - **REST API** требует больше ресурсов для обработки HTTP-запросов и ответов, что может быть неэффективно для устройств с ограниченными возможностями.

 **Масштабируемость**:
   - **MQTT** поддерживает модель "издатель-подписчик" (publish-subscribe), что позволяет легко масштабировать систему, добавляя новые устройства и клиенты без необходимости изменения архитектуры.
   - **REST API** требует точечного взаимодействия между клиентом и сервером, что может усложнить масштабирование системы.

 **Надежность и качество обслуживания (QoS)**:
   - **MQTT** поддерживает различные уровни качества обслуживания (QoS), что позволяет гарантировать доставку сообщений даже при ненадежных сетевых соединениях.
   - **REST API** не имеет встроенных механизмов для обеспечения надежной доставки сообщений, что может быть критично для IoT-приложений.

 **Реальное время**:
   - **MQTT** обеспечивает передачу сообщений в реальном времени, что важно для приложений, требующих мгновенной реакции на изменения состояния устройств.
   - **REST API** обычно используется для запросов и ответов, что может вносить задержки и не подходит для приложений реального времени.

 **Энергоэффективность**:
   - **MQTT** позволяет устройствам оставаться в режиме низкого энергопотребления, так как они могут подписываться на топики и получать сообщения только при необходимости.
   - **REST API** требует постоянного опроса сервера для получения обновлений, что увеличивает энергопотребление.

 **Гибкость и простота интеграции**:
   - **MQTT** легко интегрируется с различными устройствами и платформами благодаря своей простоте и поддержке различных форматов сообщений.
   - **REST API** может требовать дополнительных усилий для интеграции с различными устройствами и системами.

Таким образом, MQTT является более подходящим протоколом для взаимодействия с IoT-устройствами благодаря своей эффективности, масштабируемости, надежности и поддержке реального времени.
