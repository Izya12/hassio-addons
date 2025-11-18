# Анализ MiOT спецификации Xiaomi Mijia S800 (MS116)

## Критическая информация из спецификации

### URN устройства
```
urn:miot-spec-v2:device:scale:0000A07D:xiaomi-ms116:1
```

### Ключевые сервисы

#### SIID 10: Online Data (Онлайн данные)
Это основной сервис для получения данных измерений в реальном времени!

| PIID | Свойство        | Тип    | Описание                           |
|------|-----------------|--------|------------------------------------|
| 1    | work-mode       | uint8  | Режим работы весов                 |
| 2    | work-status     | uint8  | Статус измерения (0-255)           |
| 3    | work-data       | uint16 | Измеренные данные (вес?)           |
| 4    | measur-result   | string | **Результаты измерений (строка!)** |

**Action AIID 1**: `set-mode` - установить режим работы

#### SIID 6: User Management (Управление пользователями)
Профили пользователей для автоматического распознавания:

| PIID | Свойство     | Тип    | Описание                    |
|------|--------------|--------|-----------------------------|
| 1    | miid         | int64  | Mi Account ID               |
| 2    | user-id      | uint8  | ID пользователя (0-255)     |
| 3    | is-owner     | uint8  | Флаг владельца              |
| 4    | user-type    | uint8  | Тип пользователя            |
| 5    | user-age     | uint8  | Возраст                     |
| 6    | sex          | uint8  | Пол (0=female, 1=male?)     |
| 7    | user-height  | uint8  | Рост (см)                   |
| 8    | user-weight  | uint16 | Вес (кг * 100?)             |
| 9    | user-bfp     | uint16 | Body Fat Percentage * 100   |
| 10   | user-time    | uint32 | Unix timestamp создания     |

#### SIID 9: Offline Data (Оффлайн данные)
Хранение измерений когда нет соединения с Mi Home:

| PIID | Свойство     | Тип    | Описание                           |
|------|--------------|--------|-----------------------------------|
| 1    | offline-data | string | Закодированные оффлайн измерения  |
| 2    | idx-string   | string | Индексы данных                    |
| 3    | data-num     | uint8  | Количество записей (0-255)        |

#### SIID 2: Scale (Основные настройки)
| PIID | Свойство              | Тип    | Описание                      |
|------|-----------------------|--------|-------------------------------|
| 1    | unit                  | uint8  | Единица измерения (0=kg, 1=lbs, 2=jin?) |
| 2    | heart-rate-detection  | bool   | Включить детекцию пульса      |
| 3    | current-time          | uint32 | Unix timestamp                |

---

## Связь MiOT и BLE-пакетов

### Гипотеза: Mi Beacon передаёт данные MiOT

Mi Beacon используется для передачи обновлений свойств MiOT через BLE advertising. Структура может быть:

```
Зашифрованный payload после расшифровки:
┌──────────┬──────────┬──────────┬─────────────────┐
│ SIID     │ PIID     │ Length   │ Value           │
│ (1 байт) │ (1 байт) │ (1 байт) │ (variable)      │
└──────────┴──────────┴──────────┴─────────────────┘
```

Например:
- SIID=10, PIID=3 (work-data) → Вес
- SIID=10, PIID=4 (measur-result) → Полные результаты измерения

---

## Анализ реальных пакетов с учётом MiOT

### Открытый пакет (0x58)
```
Hex: 5859e25109763fcd8a43d4443d88570000000332 5d30
                              ^^^^^ Object ID?

Возможная интерпретация:
- 0x443D может быть закодированным SIID/PIID
- 0x44 = 68 decimal → не совпадает с SIID
- 0x3D = 61 decimal → не совпадает с PIID

Альтернатива - это Mi Beacon Object ID для "измерение весов"
```

### Зашифрованный пакет (0x48)
```
Hex: 4859e2510a9ec3045df18aefdec883fda3000000bac0767f

После расшифровки должны быть данные MiOT:
- SIID 10, PIID 3: work-data (uint16) - вес * 100?
- SIID 10, PIID 4: measur-result (string) - JSON с полными данными?
```

---

## Новый подход к расшифровке

### Проблема с предыдущими попытками
1. Использовали стандартную формулу nonce из ble_monitor
2. Но S800 может использовать специфичную для MiOT структуру

### Альтернативные формулы nonce для MiOT

#### Вариант 1: MiOT BLE Nonce (по спецификации Xiaomi)
```python
# Для некоторых MiOT устройств:
nonce = Device_ID (4 байта) + Frame_Counter (4 байта) + Product_ID (2 байта) + Reserved (2 байта)
```

#### Вариант 2: Упрощённая MiOT структура
```python
# Device ID может быть в данных
device_id = bytes.fromhex("763fcd8a43d4")  # MAC reversed
frame_counter = data[4]  # 0x0A
nonce = device_id + frame_counter.to_bytes(1, 'little') + b'\x00' * 5
```

#### Вариант 3: Анализ структуры пакета S800

Из реального пакета:
```
48 59 e2 51 0a 9e c3 04 5d f1 8a ef de c8 83 fd a3 00 00 00 ba c0 76 7f
│  │  │  │  │  └────────────────┘ └─────────────────────┘ └────────┘ └────┘
│  │  │  │  │     Nonce data?         Ciphertext?          Reserved   Tag
│  │  │  │  └─ Frame Counter (0x0A)
│  └──┴──┘ Product ID? (0x59E2, 0x51??)
└─ Frame Control (0x48 = encrypted)
```

Возможная структура nonce:
```python
# Байты 5-10 могут быть частью nonce
nonce_from_packet = data[5:11]  # 9e c3 04 5d f1 8a
mac_reversed = MAC[::-1]  # 76 3f cd 8a 43 d4

# Попробовать разные комбинации
```

---

## План действий для успешной расшифровки

### Шаг 1: Тестирование альтернативных nonce

Создать скрипт для перебора возможных структур nonce:

```python
from Crypto.Cipher import AES

BINDKEY = bytes.fromhex("4a4c63084637be44309909a88f7e6dc3")
MAC = bytes.fromhex("d4438acd3f76")

data_hex = "4859e2510a9ec3045df18aefdec883fda3000000bac0767f"
data = bytes.fromhex(data_hex)

# Вариант 1: Стандартный Mi Beacon v4/v5
mac_rev = MAC[::-1]
nonce1 = mac_rev + data[6:9] + data[-7:-4]

# Вариант 2: Использовать байты 5-10 как nonce data
nonce2 = mac_rev + data[5:11]

# Вариант 3: Product ID в начале
product_id = data[1:3]
frame_counter = data[4]
nonce3 = mac_rev + product_id + frame_counter.to_bytes(1, 'little') + b'\x00' * 3

# Вариант 4: Полностью из пакета
nonce4 = data[5:11] + mac_rev

# Вариант 5: Frame counter расширенный
nonce5 = mac_rev + data[4:8] + b'\x00\x00'

# Попробовать все варианты с разными offset'ами
for nonce_variant, nonce in [
    ("v1_standard", nonce1),
    ("v2_bytes_5_11", nonce2),
    ("v3_with_pid", nonce3),
    ("v4_reversed", nonce4),
    ("v5_extended_fc", nonce5)
]:
    for i in range(5, 15):  # offset от 5 до 14
        for tag_offset in [-7, -8, -6]:
            try:
                ciphertext = data[i:tag_offset]
                tag = data[-4:]
                
                cipher = AES.new(BINDKEY, AES.MODE_CCM, nonce=nonce[:12], mac_len=4)
                cipher.update(b"\x11")
                
                plaintext = cipher.decrypt_and_verify(ciphertext, tag)
                print(f"✅ SUCCESS: {nonce_variant}, offset={i}, tag_offset={tag_offset}")
                print(f"   Nonce: {nonce[:12].hex()}")
                print(f"   Plain: {plaintext.hex()}")
                break
            except:
                pass
```

### Шаг 2: Использование открытых пакетов (0x58)

Если расшифровка не удастся, можно извлечь данные из открытых пакетов:

```python
# Пакет: 5859e25109763fcd8a43d4443d88570000000332 5d30
#                                  ^^^^^^^^^^^^^^^^ Payload
payload = bytes.fromhex("88570000000332 5d30".replace(" ", ""))

# Попробовать интерпретировать как:
# - Вес (uint16): 0x5788 = 22408 → 224.08 кг? или 0x8857 = 34903 → 349.03 кг?
# - С делителем: 22408 / 100 = 224.08 или / 200 = 112.04 кг?
```

### Шаг 3: GATT Connection для MiOT

Возможно, для S800 нужно подключиться через GATT и читать характеристики MiOT:

```python
async with BleakClient(MAC_ADDRESS) as client:
    # Искать сервисы MiOT
    for service in client.services:
        if "fe95" in service.uuid or "miot" in str(service.description).lower():
            print(f"MiOT Service: {service.uuid}")
            
            for char in service.characteristics:
                if "read" in char.properties:
                    value = await client.read_gatt_char(char.uuid)
                    print(f"Char {char.uuid}: {value.hex()}")
```

---

## Дополнительные ресурсы

### MiOT BLE Спецификация
- Документация Xiaomi IoT: https://iot.mi.com/new/doc/embedded-development/ble/object-definition
- MiOT Object Format: `SIID:PIID:Value`

### Известные Object ID для весов
Из анализа других устройств Xiaomi:
- `0x0607`: Weight (вес)
- `0x0A06`: Impedance (импеданс)
- `0x1006`: Weight removed (вес удалён)

Для S800 (MiOT):
- Возможно используются пары SIID:PIID вместо Object ID
- SIID=10, PIID=3 → work-data (вес)
- SIID=10, PIID=4 → measur-result (полные данные)

---

## Следующие шаги

1. ✅ Получена спецификация MiOT
2. ⏳ Протестировать альтернативные формулы nonce
3. ⏳ Попробовать GATT connection для чтения MiOT свойств
4. ⏳ Проанализировать открытые пакеты (0x58) для извлечения веса
5. ⏳ Создать гибридный подход: открытые пакеты для веса + зашифрованные для импеданса

---

**Вывод**: Наличие спецификации MiOT - это прорыв! Теперь мы знаем структуру данных и можем целенаправленно искать способ их извлечения из BLE-пакетов.
