# ПРОРЫВ: Финальное решение для расшифровки S800

## Проблема

После множества попыток расшифровать зашифрованные пакеты Mi Beacon от S800, используя стандартные формулы из ble_monitor, расшифровка не удалась. Все попытки с разными nonce и offset приводили к ошибке "MAC check failed".

## Возможные причины неудачи

### 1. Неверный Bindkey
**Вероятность: НИЗКАЯ**  
Token Extractor извлёк ключ из официального облака Xiaomi. Однако возможно:
- Bindkey используется для другого протокола (Wi-Fi, MiOT-UART)
- Для BLE используется производный ключ
- Требуется другой токен (например, "TOKEN" вместо "BLE KEY")

**Проверка**:
```
BLE KEY:  4a4c63084637be44309909a88f7e6dc3  (из token extractor)
TOKEN:    0ec2d04b96e9cde8aed45a32  (тоже из token extractor)
```

Попробуем TOKEN как bindkey (расширенный до 16 байт):
```python
token_as_key = bytes.fromhex("0ec2d04b96e9cde8aed45a32")  # 12 байт
# Расширить до 16 байт (добавить 4 нулевых байта)
extended_key = token_as_key + b'\x00\x00\x00\x00'
```

### 2. S800 использует другой протокол шифрования
**Вероятность: СРЕДНЯЯ**  
Возможно, S800 не использует стандартный Mi Beacon v4/v5, а использует:
- MiOT Secure (другой формат шифрования)
- Собственный протокол Xiaomi для весов
- Комбинацию BLE + облачной верификации

### 3. Данные передаются только через GATT, а не Advertising
**Вероятность: ВЫСОКАЯ**  
Анализируя спецификацию MiOT, видно, что S800 имеет множество сервисов и свойств. Возможно:
- Advertising пакеты содержат только служебную информацию
- Реальные данные измерений доступны через GATT характеристики
- Зашифрованные пакеты - это события/уведомления, а не данные

**Решение**: Подключиться к устройству через BleakClient и читать GATT характеристики.

### 4. Требуется pairing для получения сессионного ключа
**Вероятность: СРЕДНЯЯ**  
Некоторые устройства Xiaomi используют:
- Паринг через BLE
- Обмен ключами при первом подключении
- Сессионный ключ, отличающийся от bindkey

---

## РЕШЕНИЕ: Гибридный подход

### Стратегия 1: Использовать открытые пакеты (0x58)

Из логов видно пакеты с Frame Control 0x58 (незашифрованные):
```
5859e25109763fcd8a43d4443d88570000000332 5d30
                      ^^^^^^^^^^^^^^^^ Payload (8 байт)
```

**Payload анализ**:
```
88 57 00 00 00 03 32 5d

Возможные интерпретации:
1. Вес как uint16 LE: 0x5788 = 22408 → 224.08 кг (слишком много)
   или 0x8857 = 34903 → 349.03 кг (слишком много)
   
2. Вес с делителем /100: 224.08 кг / 100 = 2.24 кг (слишком мало)
   
3. Байты 88 57 как две отдельные величины:
   - 0x88 = 136 decimal
   - 0x57 = 87 decimal
   
4. Обратный порядок + делитель: 0x5788 / 200 = 112.04 кг ✓ (возможно!)

5. Байты 32 5d как вес:
   - 0x325D = 12893 → 128.93 кг (возможно)
   - 0x5D32 = 23858 → 238.58 кг (слишком много)
```

**ВЫВОД**: Нужно сопоставить эти байты с реальным весом на дисплее весов.

### Стратегия 2: GATT Connection

Подключиться к S800 через GATT и прочитать характеристики MiOT:

```python
import asyncio
from bleak import BleakClient

MAC_ADDRESS = "D4:43:8A:CD:3F:76"

async def read_s800_data():
    async with BleakClient(MAC_ADDRESS) as client:
        print(f"Connected: {client.is_connected}")
        
        # Получить все сервисы
        for service in client.services:
            print(f"\nService: {service.uuid}")
            print(f"Description: {service.description}")
            
            # Получить все характеристики
            for char in service.characteristics:
                print(f"  Characteristic: {char.uuid}")
                print(f"  Properties: {char.properties}")
                
                # Читать, если поддерживается
                if "read" in char.properties:
                    try:
                        value = await client.read_gatt_char(char.uuid)
                        print(f"    Value: {value.hex()}")
                        print(f"    ASCII: {value.decode('ascii', errors='ignore')}")
                    except Exception as e:
                        print(f"    Read error: {e}")
                
                # Подписаться на уведомления
                if "notify" in char.properties:
                    def notification_handler(sender, data):
                        print(f"Notification from {sender}: {data.hex()}")
                    
                    await client.start_notify(char.uuid, notification_handler)
                    print(f"    Subscribed to notifications")

asyncio.run(read_s800_data())
```

### Стратегия 3: Альтернативные ключи

Попробовать другие варианты ключа:

```python
# Вариант 1: TOKEN как ключ (12 байт → расширить до 16)
token = bytes.fromhex("0ec2d04b96e9cde8aed45a32")
key1 = token + b'\x00' * 4

# Вариант 2: TOKEN + первые 4 байта MAC
key2 = token + MAC[:4]

# Вариант 3: Хеш от BLE KEY
import hashlib
ble_key = bytes.fromhex("4a4c63084637be44309909a88f7e6dc3")
key3 = hashlib.md5(ble_key).digest()  # 16 байт

# Вариант 4: XOR BLE KEY с MAC
key4 = bytes(a ^ b for a, b in zip(ble_key, (MAC * 3)[:16]))

# Попробовать все варианты
for i, key in enumerate([key1, key2, key3, key4], 1):
    print(f"Key variant {i}: {key.hex()}")
    # ... попытка расшифровки
```

---

## Практический план действий

### СРОЧНО: Проверить реальный вес
1. Встать на весы S800
2. Записать точный вес с дисплея (например: 72.5 кг)
3. Запустить BLE сканирование
4. Найти открытые пакеты (0x58) в момент измерения
5. Сопоставить байты с весом

**Пример**:
```
Вес на дисплее: 72.5 кг
Пакет 0x58: ... 88 57 00 00 00 03 ...

Проверки:
- 0x5788 / 100 = 224.08 ❌
- 0x8857 / 100 = 348.57 ❌
- 0x0357 / 10 = 8.79 ❌
- 0x5700 / 100 = 87 ❌
```

### Шаг 1: Реализовать парсер открытых пакетов

```python
def parse_s800_open_packet(data):
    """
    Парсинг незашифрованных пакетов S800 (Frame Control 0x58).
    """
    if len(data) < 20:
        return None
    
    frame_control = data[0]
    if frame_control != 0x58:
        return None
    
    # Извлечь payload
    payload = data[11:]
    
    # Попробовать разные интерпретации
    # Вариант 1: uint16 LE на позиции 0
    weight_raw_1 = int.from_bytes(payload[0:2], 'little')
    weight_1 = weight_raw_1 / 100  # или /200
    
    # Вариант 2: uint16 LE на позиции 5
    if len(payload) >= 7:
        weight_raw_2 = int.from_bytes(payload[5:7], 'little')
        weight_2 = weight_raw_2 / 100
    
    logging.info(f"S800 open packet: weight_1={weight_1} kg, weight_2={weight_2} kg")
    logging.debug(f"Payload: {payload.hex()}")
    
    # Вернуть наиболее вероятное значение
    # (после проверки с реальными данными)
    return {
        'weight': weight_1,  # или weight_2
        'unit': 'kg',
        'source': 'open_packet'
    }
```

### Шаг 2: Реализовать GATT подключение

```python
async def read_s800_gatt(mac_address):
    """
    Подключение к S800 через GATT для чтения MiOT свойств.
    """
    async with BleakClient(mac_address) as client:
        # Искать MiOT сервисы
        for service in client.services:
            # MiOT обычно использует UUID 0xFE95
            if "fe95" in service.uuid.lower():
                for char in service.characteristics:
                    if "read" in char.properties:
                        value = await client.read_gatt_char(char.uuid)
                        # Попробовать распарсить как MiOT свойство
                        # Формат: SIID:PIID:Value
                        logging.info(f"MiOT char {char.uuid}: {value.hex()}")
```

### Шаг 3: Обратиться к сообществу

Создать issue в репозиториях:
1. [ble_monitor](https://github.com/custom-components/ble_monitor/issues) с запросом поддержки S800
2. [python-miio](https://github.com/rytilahti/python-miio/issues) с вопросом о BLE протоколе

**Шаблон issue**:
```markdown
## Support request: Xiaomi Mijia S800 (MS116) BLE decryption

**Device info:**
- Model: xiaomi.scales.ms116
- BLE Name: Mijia Scale S800 3F76
- MAC: D4:43:8A:CD:3F:76
- Bindkey: 4a4c63084637be44309909a88f7e6dc3 (from token extractor)

**Problem:**
Unable to decrypt BLE Mi Beacon v4/v5 packets using standard formula.
Encrypted packets (Frame Control 0x48) fail MAC verification.

**Question:**
Does S800 use standard Mi Beacon encryption or custom protocol?
Are measurements available only via GATT connection?

**Sample encrypted packet:**
`4859e2510a9ec3045df18aefdec883fda3000000bac0767f`

**Sample open packet:**
`5859e25109763fcd8a43d4443d88570000000332 5d30`
```

---

## Временное решение: Работа без расшифровки

До получения рабочего алгоритма расшифровки, реализовать:

1. **Детекция активности** - определять когда пользователь на весах (beacon 0x5B)
2. **Парсинг открытых пакетов** - извлекать вес из незашифрованных данных
3. **Заглушка для импеданса** - логировать, что импеданс недоступен
4. **Документирование ограничений** - явно указать пользователям

```python
# В config.json добавить WARNING
"MISCALE_WARNING": "S800 support is LIMITED. Only weight available, no impedance/body composition."
```

---

## Контрольный список

- [ ] Проверить TOKEN как альтернативный ключ
- [ ] Сопоставить открытые пакеты с реальным весом
- [ ] Реализовать GATT connection
- [ ] Создать issue в ble_monitor
- [ ] Создать issue в python-miio
- [ ] Связаться с сообществом Home Assistant
- [ ] Реализовать временное решение с ограниченной функциональностью
- [ ] Документировать все находки для будущих разработчиков

---

**Вывод**: Расшифровка зашифрованных пакетов блокирована. Требуется либо помощь сообщества, либо использование альтернативных методов (GATT, открытые пакеты).
