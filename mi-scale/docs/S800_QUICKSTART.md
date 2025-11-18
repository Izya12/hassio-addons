# Быстрый старт: Xiaomi Mijia S800 в Home Assistant

> **Статус**: 🚧 В разработке — требуется получение bindkey для полной функциональности

## Что нужно знать

Весы **Xiaomi Mijia Body Composition Scale S800 (MJTZC04YM)** используют **зашифрованный протокол Mi Beacon**. Для получения данных о весе и составе тела необходим **bindkey** — ключ шифрования устройства.

### Возможности

| Функция | Без bindkey | С bindkey |
|---------|-------------|-----------|
| Обнаружение устройства | ✅ Да | ✅ Да |
| Определение активности | ✅ Да | ✅ Да |
| Получение веса | ❌ Нет | ✅ Да |
| Получение импеданса | ❌ Нет | ✅ Да |
| Расчёт метрик тела | ❌ Нет | ✅ Да |

---

## Шаг 1: Получите Bindkey

Следуйте **детальному руководству**: [Руководство по получению Bindkey](s800_bindkey_extraction_guide.md)

**Быстрый способ** (рекомендуется):

1. Установите `python-miio`:
   ```bash
   pip3 install python-miio
   ```

2. Войдите в Mi Cloud:
   ```bash
   miio cloud login
   ```

3. Получите список устройств:
   ```bash
   miio cloud list
   ```

4. Найдите S800 (модель `xiaomi.scales.ms116`) и скопируйте `Token` (это ваш bindkey)

**Формат bindkey**: 32 шестнадцатеричных символа, например:
```
a1b2c3d4e5f6789012345678abcdef00
```

---

## Шаг 2: Найдите MAC-адрес весов

### Способ 1: Через Mi Home

1. Откройте **Mi Home**
2. Выберите устройство **S800**
3. Нажмите на шестерёнку (настройки)
4. Пролистайте вниз до раздела **Информация о устройстве**
5. MAC-адрес отображается в формате: `D4:43:8A:CD:3F:76`

### Способ 2: Через Bluetooth сканер

1. Установите приложение **nRF Connect** (Android/iOS)
2. Начните сканирование BLE устройств
3. Встаньте на весы для активации
4. Найдите устройство с именем `Mijia Scale S800 XXXX`
5. MAC-адрес отображается под именем

---

## Шаг 3: Установите Add-On (когда будет доступна поддержка S800)

### Добавление репозитория

1. Откройте **Home Assistant**
2. Перейдите в **Settings** → **Add-ons** → **Add-on Store**
3. Нажмите три точки (⋮) → **Repositories**
4. Добавьте URL:
   ```
   https://github.com/lolouk44/hassio-addons
   ```
5. Нажмите **Add** → **Close**

### Установка Add-On

1. В Add-on Store найдите **Xiaomi Mi Scale**
2. Нажмите **Install**
3. Дождитесь завершения установки

---

## Шаг 4: Настройте Add-On

### Минимальная конфигурация

Перейдите на вкладку **Configuration** и укажите:

```yaml
HCI_DEV: hci0
MISCALE_MAC: "D4:43:8A:CD:3F:76"  # Ваш MAC-адрес
MISCALE_MODEL: s800
MISCALE_BINDKEY: "a1b2c3d4e5f6789012345678abcdef00"  # Ваш bindkey

MQTT_HOST: "127.0.0.1"
MQTT_PORT: 1883
MQTT_USERNAME: "mqtt_user"
MQTT_PASSWORD: "mqtt_password"
MQTT_PREFIX: "miscale"
MQTT_DISCOVERY: true
MQTT_DISCOVERY_PREFIX: "homeassistant"

DEBUG_LEVEL: "INFO"

USERS:
  - NAME: "Иван"
    SEX: "male"
    GT: 60
    LT: 100
    HEIGHT: 175
    DOB: "1990-01-01"
  - NAME: "Мария"
    SEX: "female"
    GT: 40
    LT: 60
    HEIGHT: 165
    DOB: "1995-05-15"
```

### Параметры пользователей

| Параметр | Описание | Пример |
|----------|----------|--------|
| `NAME` | Имя пользователя | "Иван" |
| `SEX` | Пол (male/female) | "male" |
| `GT` | Вес больше чем (Greater Than) | 60 |
| `LT` | Вес меньше чем (Less Than) | 100 |
| `HEIGHT` | Рост в см | 175 |
| `DOB` | Дата рождения (YYYY-MM-DD) | "1990-01-01" |

**Важно**: Диапазоны веса (GT/LT) не должны пересекаться между пользователями!

---

## Шаг 5: Запустите Add-On

1. Перейдите на вкладку **Info**
2. Включите **Start on boot** (опционально)
3. Нажмите **Start**
4. Перейдите на вкладку **Log** для мониторинга

### Ожидаемый лог при успешном запуске

```log
[10:00:00] INFO: Starting Xiaomi mi Scale v0.3.7...
[10:00:00] INFO: Loading Config From Options.json...
[10:00:00] DEBUG: MISCALE_MAC read from config: D4:43:8A:CD:3F:76
[10:00:00] DEBUG: MISCALE_MODEL read from config: s800
[10:00:00] DEBUG: MISCALE_BINDKEY configured: Yes
[10:00:00] INFO: Config Loaded...
[10:00:00] INFO: MQTT Discovery Setup Completed...
[10:00:05] INFO: Initialization completed, step on scale to get measurements...
```

---

## Шаг 6: Выполните измерение

1. **Встаньте на весы босиком**
2. Дождитесь стабилизации веса (дисплей перестанет мигать)
3. Останьтесь на весах ещё 3-5 секунд для измерения импеданса
4. Сойдите с весов

### Ожидаемый лог при измерении

```log
[10:05:00] DEBUG: S800 found, processing advertising data...
[10:05:02] DEBUG: S800 encrypted packet detected, decrypting...
[10:05:02] INFO: S800 measurement: weight=72.5 kg, impedance=520 Ω
[10:05:02] INFO: Matched user: Иван
[10:05:02] INFO: Publishing data to topic miscale/Иван/weight
[10:05:02] INFO: Data Published successfully
```

---

## Шаг 7: Настройте Home Assistant

### Автоматическое обнаружение

После первого измерения в **Home Assistant** автоматически появится сенсор:
- **Имя**: `sensor.иван_weight`
- **Значение**: вес в кг
- **Атрибуты**: BMI, импеданс, жировая масса, мышечная масса, и т.д.

### Ручная настройка (опционально)

Добавьте в `configuration.yaml`:

```yaml
mqtt:
  sensor:
    - name: "Иван Вес"
      state_topic: "miscale/Иван/weight"
      value_template: "{{ value_json.weight }}"
      unit_of_measurement: "kg"
      json_attributes_topic: "miscale/Иван/weight"
      icon: mdi:scale-bathroom
      state_class: measurement

    - name: "Иван BMI"
      state_topic: "miscale/Иван/weight"
      value_template: "{{ value_json.bmi }}"
      unit_of_measurement: "kg/m²"
      icon: mdi:human
      state_class: measurement

    - name: "Иван Процент жира"
      state_topic: "miscale/Иван/weight"
      value_template: "{{ value_json.body_fat }}"
      unit_of_measurement: "%"
      icon: mdi:gauge
      state_class: measurement
```

### Пример Lovelace карточки

```yaml
type: entities
title: Весы Xiaomi S800
entities:
  - entity: sensor.иван_weight
    name: Вес
  - entity: sensor.иван_bmi
    name: Индекс массы тела
  - entity: sensor.иван_body_fat
    name: Процент жира
  - type: attribute
    entity: sensor.иван_weight
    attribute: impedance
    name: Импеданс
  - type: attribute
    entity: sensor.иван_weight
    attribute: muscle_mass
    name: Мышечная масса
  - type: attribute
    entity: sensor.иван_weight
    attribute: water
    name: Процент воды
```

---

## Troubleshooting

### Проблема: "MISCALE_BINDKEY not configured"

**Решение**: Убедитесь, что в конфигурации указан параметр `MISCALE_BINDKEY` с валидным ключом (32 hex символа).

### Проблема: "Failed to decrypt S800 data"

**Решение**:
1. Проверьте правильность bindkey (скопируйте заново)
2. Убедитесь, что bindkey соответствует именно этому устройству (MAC)
3. Попробуйте переполучить bindkey через другой метод
4. Проверьте формат bindkey: `[0-9a-fA-F]{32}`

### Проблема: Устройство не обнаруживается

**Решение**:
1. Убедитесь, что MAC-адрес указан правильно
2. Проверьте, что Bluetooth адаптер работает: `hciconfig hci0 up`
3. Встаньте на весы для активации передачи BLE
4. Проверьте расстояние до весов (<5 метров)
5. Установите `DEBUG_LEVEL: "DEBUG"` и проверьте логи

### Проблема: Измерение не публикуется в MQTT

**Решение**:
1. Проверьте настройки MQTT (хост, порт, логин, пароль)
2. Убедитесь, что MQTT broker запущен
3. Проверьте, что вес попадает в диапазон GT-LT пользователя
4. Посмотрите логи на наличие ошибок MQTT

### Проблема: Импеданс не передаётся

**Решение**:
1. Убедитесь, что стоите на весах **босиком**
2. Увлажните ступни (слегка влажная кожа улучшает проводимость)
3. Стойте на весах не менее 5 секунд после стабилизации
4. Проверьте, что bindkey настроен правильно

---

## Полезные ссылки

- 📖 [Полная документация интеграции S800](xiaomi_mijia_s800_integration.md)
- 🔑 [Руководство по получению Bindkey](s800_bindkey_extraction_guide.md)
- 🔬 [Анализ реальных BLE-данных](s800_real_data_analysis.md)
- 🧪 [Руководство по исследованию протокола](s800_ble_protocol_research_guide.md)
- 💻 [Техническая спецификация реализации](s800_implementation_specification.md)

---

## Поддержка

При возникновении проблем:

1. ✅ Проверьте раздел **Troubleshooting** выше
2. 📋 Установите `DEBUG_LEVEL: "DEBUG"` и изучите логи
3. 🔍 Найдите похожие issue в [GitHub репозитории](https://github.com/lolouk44/hassio-addons/issues)
4. 🆕 Создайте новый issue с:
   - Версией add-on
   - Логами (без bindkey!)
   - Описанием проблемы
   - Шагами для воспроизведения

---

## Конфиденциальность

⚠️ **Важно**: При публикации логов или конфигураций всегда удаляйте:
- `MISCALE_BINDKEY`
- `MQTT_PASSWORD`
- Точный MAC-адрес (можно заменить на `XX:XX:XX:XX:XX:XX`)

---

> **Статус разработки**: Поддержка S800 находится в стадии активной разработки. Следите за обновлениями в репозитории проекта.
