# Техническая спецификация реализации поддержки Xiaomi Mijia S800

## Назначение документа

Настоящая спецификация предназначена для разработчиков, которые будут непосредственно реализовывать поддержку **XIAOMI MIJIA Body Composition Scale S800 (MJTZC04YM)** в коде add-on. Документ содержит:
- Детальное описание изменений кода.
- Примеры функций парсинга.
- Тестовые векторы и unit-тесты.
- Схемы обработки ошибок.
- Требования к обратной совместимости.

---

## 1. Обзор изменений в архитектуре

### 1.1. Цели
- Добавить поддержку S800 с минимальными изменениями существующего кода.
- Обеспечить автоматическое определение модели по формату пакета.
- Сохранить совместимость с Mi Scale V1 (XMTZC04HM) и V2 (XMTZC02HM, XMTZC05HM).

### 1.2. Компоненты, требующие модификации

| Файл | Изменения | Приоритет |
|------|-----------|-----------|
| `Xiaomi_Scale.py` | Добавить парсер для S800, рефакторинг `callback` | Высокий |
| `config.json` | Опционально добавить параметр `MISCALE_MODEL` | Средний |
| `Xiaomi_Scale_Body_Metrics.py` | Проверить граничные значения веса/импеданса | Низкий |
| `DOCS.md` | Обновить список поддерживаемых устройств | Высокий |
| `CHANGELOG.md` | Добавить запись о поддержке S800 | Высокий |

---

## 2. Модификация Xiaomi_Scale.py

### 2.1. Рефакторинг функции callback

**Текущий код** (упрощённый):
```python
def callback(device, advertising_data):
    global OLD_MEASURE
    if device.address.lower() == MISCALE_MAC:
        try:
            # V2 Scale
            data = binascii.b2a_hex(advertising_data.service_data['0000181b-...']).decode('ascii')
            # ... парсинг ...
        except:
            pass
        try:
            # V1 Scale
            data = binascii.b2a_hex(advertising_data.service_data['0000181d-...']).decode('ascii')
            # ... парсинг ...
        except:
            pass
```

**Проблемы**:
- Дублирование логики.
- Отсутствие явной идентификации модели.
- Трудности добавления новых форматов.

**Предлагаемый рефакторинг**:
```python
def callback(device, advertising_data):
    global OLD_MEASURE
    if device.address.lower() == MISCALE_MAC:
        measurement = parse_scale_data(advertising_data)
        if measurement:
            if should_publish(measurement, OLD_MEASURE):
                OLD_MEASURE = measurement['unique_id']
                MQTT_publish(
                    measurement['weight'],
                    measurement['unit'],
                    measurement['timestamp'],
                    measurement['has_impedance'],
                    measurement['impedance']
                )
```

### 2.2. Функция parse_scale_data

Единая функция парсинга с определением модели:
```python
def parse_scale_data(advertising_data):
    """
    Парсит BLE advertising data и возвращает словарь с измерением.
    
    Returns:
        dict или None: {
            'model': 'v2' | 'v1' | 's800',
            'weight': float,
            'unit': 'kg' | 'lbs' | 'jin',
            'has_impedance': bool,
            'impedance': int,
            'is_stabilized': bool,
            'timestamp': str (ISO 8601),
            'unique_id': float (для фильтрации дублей)
        }
    """
    # Попытка парсинга V2
    result = parse_v2_scale(advertising_data)
    if result:
        return result
    
    # Попытка парсинга S800
    result = parse_s800_scale(advertising_data)
    if result:
        return result
    
    # Попытка парсинга V1
    result = parse_v1_scale(advertising_data)
    if result:
        return result
    
    return None
```

### 2.3. Функция parse_s800_scale

Парсер для S800 (пример на основе предполагаемого формата):
```python
def parse_s800_scale(advertising_data):
    """
    Парсит данные S800 (предположительный формат).
    
    UUID: 0000181b-0000-1000-8000-00805f9b34fb (может отличаться)
    Формат: см. документацию по исследованию BLE-протокола.
    """
    SERVICE_UUID_S800 = '0000181b-0000-1000-8000-00805f9b34fb'  # Уточнить!
    
    try:
        raw_data = advertising_data.service_data.get(SERVICE_UUID_S800)
        if not raw_data:
            return None
        
        data_hex = binascii.b2a_hex(raw_data).decode('ascii')
        logging.debug(f"S800 raw data: {data_hex}")
        
        # Префикс для идентификации S800 (требуется определить)
        # Например, если S800 использует префикс 0x1B1A вместо 0x1B18
        if not data_hex.startswith('1b1a'):  # Условие для примера
            return None
        
        # Парсинг байтов
        data_bytes = bytes.fromhex(data_hex)
        
        # Control Byte 1 (смещение 2)
        ctrl_byte1 = data_bytes[2]
        is_stabilized = bool(ctrl_byte1 & (1 << 5))
        has_impedance = bool(ctrl_byte1 & (1 << 1))
        
        # Единицы измерения (смещение 4)
        meas_unit_byte = data_bytes[4]
        unit = decode_unit(meas_unit_byte)
        
        # Импеданс (смещение 9-10, little-endian)
        impedance = int.from_bytes(data_bytes[9:11], byteorder='little')
        
        # Вес (смещение 11-12, little-endian)
        weight_raw = int.from_bytes(data_bytes[11:13], byteorder='little')
        weight = weight_raw * 0.01
        
        # Корректировка для kg (возможно деление на 2, требуется проверка)
        if unit == 'kg':
            weight = weight / 2
        
        timestamp = datetime.now().strftime('%Y-%m-%dT%H:%M:%S+00:00')
        
        return {
            'model': 's800',
            'weight': round(weight, 2),
            'unit': unit,
            'has_impedance': has_impedance,
            'impedance': str(impedance) if has_impedance else '',
            'is_stabilized': is_stabilized,
            'timestamp': timestamp,
            'unique_id': round(weight, 2) + (int(impedance) if has_impedance else 0)
        }
    
    except Exception as e:
        logging.debug(f"Failed to parse S800 data: {e}")
        return None


def decode_unit(byte_value):
    """Декодирует байт единицы измерения."""
    if byte_value == 0x03:
        return 'lbs'
    elif byte_value == 0x02:
        return 'kg'
    elif byte_value == 0x12:
        return 'jin'
    else:
        logging.warning(f"Unknown unit byte: {hex(byte_value)}")
        return 'kg'  # По умолчанию
```

### 2.4. Функция should_publish

Фильтрация повторных измерений:
```python
def should_publish(measurement, old_measure):
    """
    Определяет, нужно ли публиковать измерение.
    
    Args:
        measurement (dict): Текущее измерение
        old_measure (float): Предыдущий unique_id
    
    Returns:
        bool: True если публиковать
    """
    # Публиковать только стабилизированные измерения
    if not measurement.get('is_stabilized', False):
        logging.debug("Measurement not stabilized, skipping")
        return False
    
    # Фильтр дублей
    if old_measure == measurement['unique_id']:
        logging.debug("Duplicate measurement detected, skipping")
        return False
    
    return True
```

### 2.5. Обновлённые парсеры V1 и V2

Приведение к единому интерфейсу:
```python
def parse_v2_scale(advertising_data):
    """Парсит данные Mi Scale V2."""
    SERVICE_UUID_V2 = '0000181b-0000-1000-8000-00805f9b34fb'
    try:
        raw_data = advertising_data.service_data.get(SERVICE_UUID_V2)
        if not raw_data:
            return None
        
        data_hex = binascii.b2a_hex(raw_data).decode('ascii')
        # Проверка префикса V2 (0x1B18)
        if not data_hex.startswith('1b18'):
            return None
        
        data = '1b18' + data_hex[4:]
        data_bytes = bytes.fromhex(data[4:])
        
        ctrl_byte1 = data_bytes[1]
        is_stabilized = bool(ctrl_byte1 & (1 << 5))
        has_impedance = bool(ctrl_byte1 & (1 << 1))
        
        meas_unit = data[4:6]
        unit = ''
        if meas_unit == '03':
            unit = 'lbs'
        elif meas_unit == '02':
            unit = 'kg'
        
        weight_raw = int(data[28:30] + data[26:28], 16)
        weight = weight_raw * 0.01
        if unit == 'kg':
            weight = weight / 2
        
        impedance = int(data[24:26] + data[22:24], 16)
        timestamp = datetime.now().strftime('%Y-%m-%dT%H:%M:%S+00:00')
        
        return {
            'model': 'v2',
            'weight': round(weight, 2),
            'unit': unit,
            'has_impedance': has_impedance,
            'impedance': str(impedance),
            'is_stabilized': is_stabilized,
            'timestamp': timestamp,
            'unique_id': round(weight, 2) + int(impedance)
        }
    except Exception as e:
        logging.debug(f"Failed to parse V2 data: {e}")
        return None


def parse_v1_scale(advertising_data):
    """Парсит данные Mi Scale V1."""
    SERVICE_UUID_V1 = '0000181d-0000-1000-8000-00805f9b34fb'
    try:
        raw_data = advertising_data.service_data.get(SERVICE_UUID_V1)
        if not raw_data:
            return None
        
        data_hex = binascii.b2a_hex(raw_data).decode('ascii')
        data = '1d18' + data_hex[4:]
        
        meas_unit = data[4:6]
        unit = ''
        if meas_unit.startswith(('03', 'a3')):
            unit = 'lbs'
        elif meas_unit.startswith(('12', 'b2')):
            unit = 'jin'
        elif meas_unit.startswith(('22', 'a2')):
            unit = 'kg'
        
        weight_raw = int(data[8:10] + data[6:8], 16)
        weight = weight_raw * 0.01
        if unit == 'kg':
            weight = weight / 2
        
        timestamp = datetime.now().strftime('%Y-%m-%dT%H:%M:%S+00:00')
        
        return {
            'model': 'v1',
            'weight': round(weight, 2),
            'unit': unit,
            'has_impedance': False,
            'impedance': '',
            'is_stabilized': True,  # V1 всегда стабилизирован
            'timestamp': timestamp,
            'unique_id': round(weight, 2)
        }
    except Exception as e:
        logging.debug(f"Failed to parse V1 data: {e}")
        return None
```

---

## 3. Обновление конфигурации (config.json)

### 3.1. Опциональный параметр MISCALE_MODEL

Позволяет явно указать модель весов (для оптимизации или отладки):
```json
{
  "options": {
    ...
    "MISCALE_MODEL": "auto"
  },
  "schema": {
    ...
    "MISCALE_MODEL": "list(auto|v1|v2|s800)?"
  }
}
```

**Использование в коде**:
```python
MISCALE_MODEL = data.get("MISCALE_MODEL", "auto")
if MISCALE_MODEL == "s800":
    # Принудительно использовать только парсер S800
    result = parse_s800_scale(advertising_data)
elif MISCALE_MODEL == "auto":
    # Автоматическое определение
    result = parse_scale_data(advertising_data)
```

---

## 4. Тестовые векторы и unit-тесты

### 4.1. Тестовые данные (пример)

```python
# tests/test_s800_parser.py
import unittest
from unittest.mock import MagicMock
import binascii

class TestS800Parser(unittest.TestCase):
    
    def test_parse_s800_kg_with_impedance(self):
        """Тест парсинга S800: кг, импеданс 500 Ом, вес 70.5 кг."""
        # Симуляция advertising_data
        # Формат: 1b1a 22 00 02 ... 01F4 0DC2
        # 0x01F4 = 500 (impedance), 0x0DC2 = 3522 → 35.22 * 2 = 70.44 кг
        hex_data = '1b1a220002000000000001f40dc20000000000000000'
        raw_data = bytes.fromhex(hex_data)
        
        advertising_data = MagicMock()
        advertising_data.service_data = {
            '0000181b-0000-1000-8000-00805f9b34fb': raw_data
        }
        
        result = parse_s800_scale(advertising_data)
        
        self.assertIsNotNone(result)
        self.assertEqual(result['model'], 's800')
        self.assertEqual(result['unit'], 'kg')
        self.assertAlmostEqual(result['weight'], 70.5, places=1)
        self.assertTrue(result['has_impedance'])
        self.assertEqual(result['impedance'], '500')
        self.assertTrue(result['is_stabilized'])
    
    def test_parse_s800_lbs_no_impedance(self):
        """Тест парсинга S800: фунты, без импеданса, вес 120 lbs."""
        # 0x03 = lbs, вес 0x04B0 = 1200 → 12.00 lbs (если масштаб *0.01)
        hex_data = '1b1a200003000000000000000bb80000000000000000'
        raw_data = bytes.fromhex(hex_data)
        
        advertising_data = MagicMock()
        advertising_data.service_data = {
            '0000181b-0000-1000-8000-00805f9b34fb': raw_data
        }
        
        result = parse_s800_scale(advertising_data)
        
        self.assertIsNotNone(result)
        self.assertEqual(result['unit'], 'lbs')
        self.assertAlmostEqual(result['weight'], 120.0, places=1)
        self.assertFalse(result['has_impedance'])
    
    def test_parse_s800_not_stabilized(self):
        """Тест: измерение не стабилизировано (bit 5 = 0)."""
        hex_data = '1b1a020002000000000001f40dc20000000000000000'
        raw_data = bytes.fromhex(hex_data)
        
        advertising_data = MagicMock()
        advertising_data.service_data = {
            '0000181b-0000-1000-8000-00805f9b34fb': raw_data
        }
        
        result = parse_s800_scale(advertising_data)
        
        self.assertIsNotNone(result)
        self.assertFalse(result['is_stabilized'])
        # should_publish вернёт False для нестабильного
    
    def test_parse_wrong_prefix(self):
        """Тест: неверный префикс (не S800)."""
        hex_data = '1b18220002000000000001f40dc20000000000000000'  # V2 префикс
        raw_data = bytes.fromhex(hex_data)
        
        advertising_data = MagicMock()
        advertising_data.service_data = {
            '0000181b-0000-1000-8000-00805f9b34fb': raw_data
        }
        
        result = parse_s800_scale(advertising_data)
        
        self.assertIsNone(result)  # Не распознан как S800

if __name__ == '__main__':
    unittest.main()
```

### 4.2. Запуск тестов

Добавить в `requirements.txt` (для разработки):
```
bleak>=0.20.0
paho-mqtt>=1.5.0
pytest>=7.0.0
pytest-cov>=3.0.0
```

Команда запуска:
```bash
cd /home/engine/project/mi-scale
python3 -m pytest tests/ -v --cov=src
```

---

## 5. Обработка ошибок и логирование

### 5.1. Уровни логирования

| Сценарий | Уровень | Сообщение (пример) |
|----------|---------|-------------------|
| Успешный парсинг S800 | DEBUG | "S800 data parsed: weight=70.5 kg, impedance=500" |
| Не распознан формат | DEBUG | "Failed to parse S800 data: KeyError at byte 11" |
| Нестабильное измерение | DEBUG | "Measurement not stabilized, skipping" |
| Дубль измерения | DEBUG | "Duplicate measurement detected, skipping" |
| Невалидные данные | WARNING | "Invalid weight value: -10 kg, ignoring" |
| Исключение в парсере | ERROR | "Exception in parse_s800_scale: {exception}" |

### 5.2. Валидация данных

Добавить проверки в парсер:
```python
# После парсинга веса
if weight < 0 or weight > 200:
    logging.warning(f"Invalid weight value: {weight} kg, ignoring")
    return None

# После парсинга импеданса
if has_impedance and (impedance < 50 or impedance > 3000):
    logging.warning(f"Invalid impedance value: {impedance} Ohm, ignoring")
    return None
```

### 5.3. Обработка неизвестных единиц

```python
def decode_unit(byte_value):
    unit_map = {
        0x02: 'kg',
        0x03: 'lbs',
        0x12: 'jin'
    }
    unit = unit_map.get(byte_value)
    if not unit:
        logging.warning(f"Unknown unit byte: {hex(byte_value)}, defaulting to kg")
        return 'kg'
    return unit
```

---

## 6. Обратная совместимость

### 6.1. Проверочный список
- [x] Парсеры V1 и V2 не изменяют публичные интерфейсы.
- [x] Функция `MQTT_publish` принимает те же параметры.
- [x] `OLD_MEASURE` корректно работает с `unique_id`.
- [x] Конфигурация `options.json` не требует обязательных новых параметров.
- [x] MQTT Discovery совместим с предыдущими версиями.

### 6.2. Тестирование с реальными устройствами
- Протестировать на Mi Scale V1 (XMTZC04HM).
- Протестировать на Mi Scale V2 (XMTZC02HM, XMTZC05HM).
- Протестировать на S800 (MJTZC04YM).

---

## 7. Документация

### 7.1. Обновление DOCS.md

Добавить в секцию "Supported Scales":
```markdown
| Name | Model | Picture |
| --- | --- | --- |
| [Mi Smart Scale 2](https://www.mi.com/global/scale) | XMTZC04HM | <img ...> |
| [Mi Body Composition Scale](https://www.mi.com/global/mi-body-composition-scale/) | XMTZC02HM | <img ...> |
| [Mi Body Composition Scale 2](https://c.mi.com/thread-2289389-1-0.html) | XMTZC05HM | <img ...> |
| **[Mijia Body Composition Scale S800](https://www.mi.com/...)** | **MJTZC04YM** | <img ...> |
```

Добавить примечание:
```markdown
### Notes for S800 users
- The S800 uses a slightly different BLE protocol. Ensure your Bluetooth adapter supports BLE 4.0+.
- If measurements are not detected, check the logs for `S800 data parsed` messages.
- For troubleshooting, set `DEBUG_LEVEL` to `DEBUG` in the add-on configuration.
```

### 7.2. Обновление CHANGELOG.md

```markdown
## [0.3.7] - 2024-03-20

### Added
- Support for XIAOMI MIJIA Body Composition Scale S800 (MJTZC04YM, xiaomi.scales.ms116).
- Refactored BLE packet parsing for better maintainability.
- Optional `MISCALE_MODEL` configuration parameter for explicit model selection.

### Changed
- Improved error handling and logging for BLE data parsing.

### Fixed
- Minor bug fixes in duplicate measurement filtering.
```

---

## 8. План развёртывания

### 8.1. Этапы
1. **Разработка**: Реализация парсера, тестирование на симулированных данных.
2. **Бета-тестирование**: Релиз pre-release версии для пользователей S800.
3. **Сбор обратной связи**: Анализ логов, корректировка парсера.
4. **Стабильный релиз**: Публикация версии 0.3.7.

### 8.2. Критерии готовности
- [ ] Успешное распознавание и парсинг S800 пакетов.
- [ ] Корректные расчёты метрик тела для 3+ пользователей.
- [ ] Публикация в MQTT без дублирования.
- [ ] Обратная совместимость с V1/V2.
- [ ] Обновлённая документация.
- [ ] Unit-тесты с покрытием >80%.

---

## 9. Контрольные точки и метрики

### 9.1. Метрики качества
- **Успешность парсинга**: >95% пакетов S800 распознаются корректно.
- **Точность веса**: отклонение от дисплея <±0.1 кг.
- **Задержка публикации**: <2 секунды от момента стабилизации.
- **Отсутствие ложных срабатываний**: 0 публикаций для нестабильных измерений.

### 9.2. Мониторинг в production
- Логировать количество успешных/неуспешных парсингов S800.
- Отслеживать жалобы пользователей на неточность данных.
- Анализировать логи DEBUG для выявления паттернов ошибок.

---

## 10. Дополнительные рекомендации

### 10.1. Оптимизация производительности
- Избегать повторных вычислений в `callback` (кеширование).
- Использовать `asyncio` эффективно, не блокировать event loop.

### 10.2. Безопасность
- Валидировать входные данные на предмет переполнений.
- Не логировать чувствительные данные (MAC-адреса в INFO/WARNING уровнях).

### 10.3. Расширяемость
- Продумать архитектуру для добавления будущих моделей (S900, S1000, etc.).
- Рассмотреть вынос парсеров в отдельные модули (например, `parsers/s800.py`).

---

## Приложения

### A. Полная структура пакета S800 (предположительная)

```
Offset | Length | Field              | Example (hex) | Description
-------+--------+--------------------+---------------+----------------------------
0      | 2      | Header             | 1B 1A         | Identifier (S800 specific)
2      | 1      | Control Byte 1     | 22            | Flags (stabilized, impedance)
3      | 1      | Control Byte 2     | 00            | Reserved
4      | 1      | Measurement Unit   | 02            | 0x02=kg, 0x03=lbs, 0x12=jin
5      | 4      | Timestamp          | XX XX XX XX   | Year/Month/Day/Hour
9      | 2      | Impedance (LE)     | F4 01         | 0x01F4 = 500 Ohm
11     | 2      | Weight (LE)        | C2 0D         | 0x0DC2 = 3522 → 35.22 kg
13     | 7      | Reserved/Extra     | ...           | Future use or checksum
```

### B. Битовые флаги Control Byte 1

```
Bit 7   | Bit 6   | Bit 5       | Bit 4    | Bit 3    | Bit 2    | Bit 1       | Bit 0
Reserved| Reserved| Stabilized  | Reserved | Reserved | Reserved | Impedance   | Reserved
        |         | (1=stable)  |          |          |          | (1=present) |
```

---

## Заключение

Настоящая спецификация предоставляет полное руководство для реализации поддержки S800 в коде add-on. После завершения исследования BLE-протокола и подтверждения формата пакетов, следует:
1. Обновить функции-заглушки (например, префикс `1b1a`, смещения байтов).
2. Запустить unit-тесты с реальными дампами.
3. Провести интеграционное тестирование с устройством S800.
4. Обновить документацию и выпустить релиз.

При возникновении вопросов или обнаружении расхождений с реальным форматом, откорректируйте данную спецификацию и оставьте комментарии в коде для будущих разработчиков.
