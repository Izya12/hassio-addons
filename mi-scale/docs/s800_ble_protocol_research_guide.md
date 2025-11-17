# Руководство по исследованию BLE-протокола XIAOMI MIJIA Body Composition Scale S800

## Введение

Данное руководство предназначено для инженеров и исследователей, которые будут проводить сбор и анализ BLE-пакетов от весов **XIAOMI MIJIA Body Composition Scale S800 (MJTZC04YM, xiaomi.scales.ms116)**. Документ описывает методологию, инструменты и последовательность действий для выявления структуры данных устройства.

---

## 1. Подготовка инструментария

### 1.1. Необходимое оборудование
- **BLE-совместимый адаптер**: Bluetooth 4.0+ (желательно Bluetooth 5.0+).
- **Устройство S800**: полностью заряженное, готовое к измерениям.
- **Хост-система**: Linux (Ubuntu, Debian, Raspberry Pi OS) или Android/iOS с приложениями для BLE.

### 1.2. Программное обеспечение

#### Linux
```bash
# Установка BlueZ и утилит
sudo apt-get update
sudo apt-get install -y bluez bluez-tools

# Проверка доступности адаптера
hciconfig
# Ожидается: hci0, UP, RUNNING

# Установка Python и Bleak
sudo apt-get install -y python3 python3-pip
pip3 install bleak
```

#### Windows
- **nRF Connect for Desktop** ([https://www.nordicsemi.com/Products/Development-tools/nrf-connect-for-desktop](https://www.nordicsemi.com/Products/Development-tools/nrf-connect-for-desktop))
- **Bluetooth HCI Sniffer** (Nordic Semiconductor)

#### Android
- **nRF Connect for Mobile** (Play Store/App Store)
- **BLE Scanner** (Play Store)

#### iOS
- **nRF Connect for Mobile** (App Store)
- **LightBlue** (App Store)

### 1.3. Рекомендуемые скрипты

#### Сканирование устройств (Python, Bleak)
```python
import asyncio
from bleak import BleakScanner

async def scan():
    devices = await BleakScanner.discover(timeout=30.0)
    for device in devices:
        if "MIBFS" in device.name or "XMTZC" in device.name or "S800" in device.name or "MI" in (device.name or "").upper():
            print(f"Device: {device.name}, Address: {device.address}")
            print(f"Metadata: {device.metadata}")
            print(f"RSSI: {device.rssi}")
            print("---")

asyncio.run(scan())
```

#### Захват advertising data (Python, Bleak)
```python
import asyncio
import binascii
from bleak import BleakScanner

TARGET_MAC = "AA:BB:CC:DD:EE:FF"  # Замените на реальный MAC S800

def callback(device, advertising_data):
    if device.address.upper() == TARGET_MAC.upper():
        print(f"[{device.address}] Local Name: {device.name}")
        print(f"[{device.address}] RSSI: {advertising_data.rssi}")
        print(f"[{device.address}] Service UUIDs: {advertising_data.service_uuids}")
        print(f"[{device.address}] Service Data:")
        for uuid, data in advertising_data.service_data.items():
            print(f"  UUID: {uuid}, Data (hex): {binascii.hexlify(data).decode('ascii')}")
        print(f"[{device.address}] Manufacturer Data:")
        for key, data in advertising_data.manufacturer_data.items():
            print(f"  Company ID: {key}, Data (hex): {binascii.hexlify(data).decode('ascii')}")
        print("=" * 80)

async def main():
    async with BleakScanner(callback) as scanner:
        await asyncio.sleep(300.0)  # Сканирование 5 минут

asyncio.run(main())
```

#### Логирование с bluetoothctl
```bash
bluetoothctl
[bluetooth]# scan on
# Встаньте на весы, дождитесь стабилизации веса
# Наблюдайте за выводом устройств, содержащих "MIBFS" или ваш MAC

# Копируйте вывод, где отображаются Service Data и Manufacturer Data
```

#### Захват с btmon
```bash
# В отдельном терминале
sudo btmon > /tmp/s800_capture.log

# В другом терминале
# Встаньте на весы и выполните измерение
# Нажмите Ctrl+C в btmon после получения данных

# Анализируйте файл /tmp/s800_capture.log
```

---

## 2. Процедура сбора данных

### 2.1. Подготовка устройства
1. Убедитесь, что весы полностью заряжены или имеют свежие батареи.
2. Отключите Mi Home и другие приложения, которые могут конкурировать за BLE-соединение (рекомендация).
3. Разместите весы на ровной поверхности.

### 2.2. Регистрация MAC-адреса
- Откройте приложение **Mi Home** → Профиль устройства → Информация.
- Запишите полный MAC-адрес (например, `AA:BB:CC:DD:EE:FF`).
- Альтернативно, используйте `bluetoothctl` и идентифицируйте устройство по названию (`MIBFS` или аналог).

### 2.3. Серия измерений для разных сценариев

Выполните следующие тестовые сценарии, записывая дамп для каждого:

| № | Сценарий | Описание |
|---|----------|----------|
| 1 | Легкий пользователь, кг | Вес ~50 кг, единицы измерения: кг, стабильное измерение |
| 2 | Тяжёлый пользователь, кг | Вес ~90 кг, единицы измерения: кг, стабильное измерение |
| 3 | Единицы: фунты | Переключите весы в режим фунтов (lb) через Mi Home, выполните измерение |
| 4 | Единицы: цзинь | Переключите в режим jin, измерение |
| 5 | Нестабильное измерение | Встаньте на весы, но не дожидайтесь стабилизации, снимите вес через 1-2 сек |
| 6 | Отсутствие пользователя | Легкий предмет (гиря 5 кг), измерение без биоимпеданса |
| 7 | Повторное измерение | Встаньте сразу после завершения предыдущего измерения (проверка дублей) |

### 2.4. Структура дампа

Для каждого сценария записывайте:
```
Дата/время: 2024-03-15 14:30:00
MAC: AA:BB:CC:DD:EE:FF
Сценарий: Легкий пользователь, кг
Вес (по дисплею весов): 52.5 кг
Импеданс (если есть в Mi Home): 500 Ом

Service UUID: 0000181b-0000-1000-8000-00805f9b34fb
Service Data (hex): 1b182200000c500fa00000...

Manufacturer Data:
Company ID: 0x038F (Xiaomi)
Data (hex): 0500...

Timestamp: <Unix timestamp из пакета, если есть>
```

### 2.5. Автоматизация сбора

Используйте Python-скрипт с выводом в CSV:
```python
import asyncio
import csv
import datetime
import binascii
from bleak import BleakScanner

TARGET_MAC = "AA:BB:CC:DD:EE:FF"
OUTPUT_FILE = "s800_measurements.csv"

measurements = []

def callback(device, advertising_data):
    if device.address.upper() == TARGET_MAC.upper():
        timestamp = datetime.datetime.now().isoformat()
        for uuid, data in advertising_data.service_data.items():
            hex_data = binascii.hexlify(data).decode('ascii')
            measurements.append({
                'timestamp': timestamp,
                'mac': device.address,
                'uuid': uuid,
                'service_data_hex': hex_data,
                'rssi': advertising_data.rssi
            })

async def main():
    async with BleakScanner(callback) as scanner:
        await asyncio.sleep(600.0)  # 10 минут

    with open(OUTPUT_FILE, 'w', newline='') as f:
        writer = csv.DictWriter(f, fieldnames=['timestamp', 'mac', 'uuid', 'service_data_hex', 'rssi'])
        writer.writeheader()
        writer.writerows(measurements)
    print(f"Measurements saved to {OUTPUT_FILE}")

asyncio.run(main())
```

---

## 3. Анализ структуры BLE-пакетов

### 3.1. Идентификация типа пакета

- Проверьте, какие UUID встречаются в `service_data`:
  - `0000181b-...`: Body Composition Service (стандартизирован).
  - `0000181d-...`: Weight Scale Service (Mi Scale V1).
  - `0000fe95-...`: Mi Beacon (MiOT, шифрование).
- Определите, какой UUID используется S800.

### 3.2. Парсинг service_data

Предположим, формат аналогичен Mi Scale V2 (`0x181B`):

```
Байты    Поле                  Пример
0-1      Header (0x1B18)       0x1B18
2        Control Byte 1        0x22
         - Bit 5: stabilized   (1 = вес стабилен)
         - Bit 1: impedance    (1 = импеданс доступен)
3        Control Byte 2        0x00
4-5      Единицы измерения     0x03 (lbs), 0x02 (kg), 0x12 (jin)
6-9      Год/Месяц/День/Час    ...
10-11    Импеданс (LE)         0x0C0E (little-endian: 0x0E0C = 3596 Ом)
12-13    Вес (LE)              0x0FA0 (little-endian: 0xA00F = 40975 → 40975 * 0.01 / 2 = 204.875 г / 2)
```

**Важно**:
- Весы могут передавать данные в **little-endian** (младший байт первым).
- Масштабирование веса: часто `value * 0.01 / 2` для kg (уточните).
- Управляющий байт содержит битовые флаги.

### 3.3. Парсинг manufacturer_data

Если присутствуют пакеты с Company ID `0x038F` (Xiaomi):
```
Байты    Поле                  Описание
0-1      Frame Control         Флаги шифрования, версии
2-3      Product ID            Идентификатор модели (ms116?)
4        Frame Counter         Счётчик пакетов
5-6      MAC Address (part)    Часть MAC-адреса
7+       Payload               Данные (вес, импеданс и т.д.)
```

MiOT-пакеты могут быть зашифрованы AES-CCM с ключом `bindkey` из Mi Home. Для получения ключа:
- Используйте **miio** или **mible** библиотеки Python.
- Экспортируйте конфигурацию из Mi Home через инструмент `python-miio` (`miio discover`, `miio-extract-tokens`).

### 3.4. Выявление битовых масок

Создайте таблицу изменений байта контроля:

| Измерение | Вес стабилен? | Импеданс есть? | Control Byte (hex) | Control Byte (bin) |
|-----------|---------------|----------------|--------------------|---------------------|
| 1         | Да            | Да             | 0x22               | 0010 0010           |
| 2         | Нет           | Да             | 0x02               | 0000 0010           |
| 3         | Да            | Нет            | 0x20               | 0010 0000           |
| 4         | Нет           | Нет            | 0x00               | 0000 0000           |

Выявите биты:
- Bit 5 (0x20): stabilized.
- Bit 1 (0x02): impedance available.
- Биты 0, 6-7: возможно, единицы измерения или резерв.

### 3.5. Масштабирование веса

Зафиксируйте реальный вес по дисплею и сравните с байтами:
```
Дисплей: 52.5 кг
Байты (LE): 0xC814 → 0x14C8 = 5320
Расчёт: 5320 * 0.01 = 53.20 кг (близко)
Уточнение: возможно, делитель зависит от единиц измерения.

Для фунтов: value * 0.01 (без делителя /2).
Для кг: value * 0.01 / 2 (или проверьте на практике).
```

### 3.6. Импеданс

Байты импеданса обычно 2 байта (LE):
```
Байты: 0x0C0E → 0x0E0C = 3596 Ом
Mi Home показывает: 359 Ом → возможно, делитель 10
```

---

## 4. Документирование результатов

### 4.1. Шаблон отчёта

```markdown
# Анализ BLE-пакетов S800

## Устройство
- Модель: MJTZC04YM (S800)
- MAC: AA:BB:CC:DD:EE:FF
- Дата исследования: 2024-03-15

## Формат пакетов
- UUID: 0000181b-0000-1000-8000-00805f9b34fb (Body Composition)
- Длина service_data: 20 байт

## Структура (байты)
| Offset | Длина | Поле               | Значение (пример) | Описание                  |
|--------|-------|--------------------|-------------------|---------------------------|
| 0      | 2     | Header             | 0x1B18            | Идентификатор формата     |
| 2      | 1     | Control Byte 1     | 0x22              | Флаги (stabilized, imp.)  |
| 3      | 1     | Control Byte 2     | 0x00              | Резерв                    |
| 4      | 1     | Measurement Unit   | 0x02              | 0x02=kg, 0x03=lbs, 0x12=jin |
| 5      | 4     | Timestamp          | ...               | Год/Месяц/День/Час        |
| 9      | 2     | Impedance (LE)     | 0x0E0C            | Импеданс (3596 Ом)        |
| 11     | 2     | Weight (LE)        | 0xC814            | Вес (5320 → 53.20 кг)     |
| 13     | 7     | Reserved/Extra     | ...               | Возможно, доп. данные     |

## Формулы
- Вес (кг): `weight_raw * 0.01 / 2`
- Импеданс (Ом): `impedance_raw`
- Единицы: 0x02 = kg, 0x03 = lbs, 0x12 = jin

## Особенности
- Устройство передаёт пакеты только при стабилизации веса (bit 5 = 1).
- Импеданс доступен только при босых ногах (bit 1 = 1).
- При повторном измерении пакет дублируется; требуется фильтрация по значению.

## Тестовые данные
См. файл `s800_measurements.csv`.
```

### 4.2. Визуализация

Создайте диаграмму байтов:
```
0  1  2  3  4  5  6  7  8  9  10 11 12 13 14 15 16 17 18 19
┌──┬──┬──┬──┬──┬─────────┬─────┬─────┬─────────────────────┐
│HD│HD│C1│C2│UN│TS       │IMP  │WT   │ RESERVED            │
└──┴──┴──┴──┴──┴─────────┴─────┴─────┴─────────────────────┘
HD = Header (0x1B18)
C1 = Control Byte 1
C2 = Control Byte 2
UN = Unit
TS = Timestamp
IMP = Impedance
WT = Weight
```

---

## 5. Частые вопросы и решения

### 5.1. Устройство не обнаруживается
- Убедитесь, что весы включены (встаньте на них).
- Проверьте, что Bluetooth адаптер работает: `sudo hciconfig hci0 up`.
- Используйте `sudo` для скриптов Bleak/bluetoothctl.

### 5.2. Пакеты не содержат service_data
- Возможно, устройство использует MiOT (`0xFE95`).
- Проверьте `manufacturer_data` с Company ID `0x038F`.
- Исследуйте шифрование MiOT.

### 5.3. Значения веса некорректны
- Проверьте порядок байтов (little-endian vs big-endian).
- Уточните масштабирование (`* 0.01`, `/ 2`, и т.д.).
- Сопоставьте с несколькими измерениями разного веса.

### 5.4. Импеданс не передаётся
- Убедитесь, что стоите босиком на весах.
- Проверьте bit 1 Control Byte (должен быть = 1).

### 5.5. Mi Home не показывает данные параллельно
- BLE-устройства часто допускают только одно активное соединение.
- Остановите Mi Home перед запуском скриптов.

---

## 6. Рекомендации по безопасности

- **Не публикуйте MiOT bind keys** устройства в открытых источниках.
- При отладке удаляйте MAC-адреса из логов перед публикацией.
- Используйте виртуальные окружения Python для изоляции зависимостей.

---

## 7. Следующие шаги после завершения исследования

1. Заполнить документ `xiaomi_mijia_s800_integration.md` фактическими данными.
2. Обновить `Xiaomi_Scale.py`: добавить обработчик нового формата.
3. Создать юнит-тесты парсинга с реальными дампами.
4. Обновить `DOCS.md`: добавить S800 в список поддерживаемых устройств.
5. Протестировать интеграцию с реальным устройством и несколькими пользователями.
6. Релиз новой версии add-on.

---

> **Контактная информация**: при возникновении вопросов или обнаружении неточностей, пожалуйста, оставьте issue в репозитории проекта.
