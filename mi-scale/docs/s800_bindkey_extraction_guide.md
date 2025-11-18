# Руководство по получению Bindkey для Xiaomi Mijia S800

## Введение

Для полноценной работы с весами **Xiaomi Mijia Body Composition Scale S800 (MJTZC04YM)** необходим **bindkey** — 128-битный ключ шифрования, используемый для расшифровки BLE-пакетов, передаваемых устройством по протоколу Mi Beacon.

Без bindkey возможны только:
- Обнаружение устройства
- Определение активности (idle/measuring)
- Получение служебной информации

С bindkey становятся доступны:
- Точный вес пользователя
- Биоимпеданс
- Единицы измерения
- Все метрики состава тела

---

## Метод 1: Извлечение через Mi Home (Android) — Рекомендуется

### Требования
- Android-устройство с Mi Home
- Устройство S800, добавленное в Mi Home
- Доступ к ADB или файловой системе (root)

### Шаг 1: Включение режима разработчика в Mi Home

1. Откройте приложение **Mi Home**
2. Перейдите в **Профиль** (вкладка справа)
3. Перейдите в **Настройки** (шестерёнка)
4. Нажмите **О приложении**
5. Быстро нажмите на **Версия** 10 раз
6. Появится уведомление: "Вы теперь разработчик!"
7. Вернитесь в Настройки → появится раздел **Дополнительные настройки** или **Developer mode**
8. Включите **Включить расширенное логирование** или **Enable Developer Mode**

### Шаг 2: Добавление устройства (если ещё не добавлено)

1. В Mi Home нажмите **+** (добавить устройство)
2. Выберите категорию **Здоровье** → **Умные весы**
3. Выберите модель **Mijia Body Composition Scale S800**
4. Встаньте на весы для активации
5. Следуйте инструкциям на экране для сопряжения
6. После успешного добавления устройство появится в списке

### Шаг 3: Извлечение токенов через ADB

**Вариант A: Через ADB (без root)**

1. Установите **Android SDK Platform Tools** (adb)
2. Включите **Отладку по USB** на Android-устройстве
3. Подключите устройство к компьютеру
4. Выполните команды:

```bash
# Проверка подключения
adb devices

# Создание backup Mi Home (без пароля)
adb backup -noapk com.xiaomi.smarthome -f mi_home_backup.ab

# Конвертация backup в tar (требуется инструмент abe.jar)
java -jar abe.jar unpack mi_home_backup.ab mi_home_backup.tar

# Извлечение tar
tar -xf mi_home_backup.tar

# Поиск базы данных
find . -name "*.db" | grep miio

# Открыть базу данных
sqlite3 apps/com.xiaomi.smarthome/db/miio2.db
```

4. В SQLite выполните:
```sql
SELECT * FROM devicerecord;
```

5. Найдите запись с моделью `xiaomi.scales.ms116` или MAC `D4:43:8A:CD:3F:76`
6. Скопируйте значение поля `token` (это bindkey в hex-формате, 32 символа)

**Вариант B: Через логи Mi Home**

1. С включённым режимом разработчика откройте Mi Home
2. Перейдите к устройству S800
3. Выполните любое действие (измерение веса)
4. Извлеките логи:

```bash
adb logcat -d > mi_home_logs.txt
```

5. Откройте `mi_home_logs.txt` и найдите строки с:
   - `token:`
   - `bindkey:`
   - `beaconkey:`

6. Ключ будет в формате: `0123456789abcdef0123456789abcdef` (32 hex символа)

### Шаг 4: Извлечение через файловую систему (root требуется)

Если у вас есть root:

```bash
# Получить shell с root
adb shell
su

# Найти базу данных Mi Home
find /data/data/com.xiaomi.smarthome -name "*.db"

# Скопировать базу на sdcard
cp /data/data/com.xiaomi.smarthome/databases/miio2.db /sdcard/

# Выйти из shell
exit

# Скачать базу на компьютер
adb pull /sdcard/miio2.db

# Открыть в SQLite Browser или командой
sqlite3 miio2.db "SELECT * FROM devicerecord WHERE model LIKE '%ms116%';"
```

---

## Метод 2: Через облачный API Xiaomi

### Требования
- Учётная запись Xiaomi
- Python 3.7+
- Библиотека `python-miio`

### Установка

```bash
pip3 install python-miio
```

### Получение токенов облака

**Вариант A: Через miio CLI**

```bash
# Логин в Mi Cloud
miio cloud login

# Введите учётные данные Xiaomi (email/телефон и пароль)
# Выберите регион (cn, de, i2, ru, sg, us)

# Получить список устройств
miio cloud list

# Найти S800 в списке (модель: xiaomi.scales.ms116)
# Bindkey будет отображён в столбце 'Token'
```

**Вариант B: Через Python скрипт**

```python
#!/usr/bin/env python3
from miio.cloud import CloudInterface
import getpass

# Учётные данные
username = input("Xiaomi username (email/phone): ")
password = getpass.getpass("Password: ")

# Регион (cn, de, i2, ru, sg, us)
country = input("Country (default: de): ") or "de"

# Подключение к облаку
cloud = CloudInterface(username, password, country=country)

# Логин
if not cloud.login():
    print("Login failed!")
    exit(1)

print("Login successful!")

# Получение устройств
devices = cloud.get_devices()

# Поиск S800
for device in devices:
    if 'ms116' in device.get('model', '') or 's800' in device.get('name', '').lower():
        print("\n=== Xiaomi Mijia S800 Found ===")
        print(f"Name: {device.get('name')}")
        print(f"Model: {device.get('model')}")
        print(f"MAC: {device.get('mac')}")
        print(f"Token/Bindkey: {device.get('token')}")
        print(f"Device ID: {device.get('did')}")
        print("================================\n")
```

Сохраните как `get_bindkey.py` и запустите:
```bash
python3 get_bindkey.py
```

---

## Метод 3: Через Xiaomi Cloud Tokens Extractor (Упрощённый)

### Инструмент онлайн

1. Перейдите на сайт: [Xiaomi Cloud Tokens Extractor](https://github.com/PiotrMachowski/Xiaomi-cloud-tokens-extractor)
2. Скачайте инструмент:
```bash
git clone https://github.com/PiotrMachowski/Xiaomi-cloud-tokens-extractor.git
cd Xiaomi-cloud-tokens-extractor
pip3 install -r requirements.txt
```

3. Запустите:
```bash
python3 token_extractor.py
```

4. Введите учётные данные Xiaomi
5. Выберите регион
6. Инструмент автоматически извлечёт токены всех устройств
7. Найдите S800 и скопируйте bindkey

---

## Метод 4: Через Mi Home Web API (Advanced)

### Ручное извлечение через браузер

1. Откройте браузер (Chrome/Firefox) с DevTools
2. Перейдите на [Mi Home Web](https://home.mi.com/)
3. Войдите в учётную запись Xiaomi
4. Откройте DevTools (F12) → вкладка **Network**
5. Обновите страницу или перейдите в раздел устройств
6. Найдите запросы к API (например, `/v2/device/list`)
7. Просмотрите ответы (Response) в формате JSON
8. Найдите устройство S800 (`ms116`) и поле `token` или `bindkey`

**Пример ответа API:**
```json
{
  "result": {
    "list": [
      {
        "did": "123456789",
        "model": "xiaomi.scales.ms116",
        "name": "Mijia Scale S800",
        "mac": "D4:43:8A:CD:3F:76",
        "token": "0123456789abcdef0123456789abcdef"
      }
    ]
  }
}
```

---

## Метод 5: BLE Sniffing (Экспертный уровень)

### Требования
- Nordic nRF52840 Dongle или аналог
- nRF Sniffer for Bluetooth LE
- Wireshark

### Процесс

1. Настройте BLE sniffer для захвата трафика на частоте 2.4 GHz
2. Сбросьте устройство S800 к заводским настройкам
3. Начните захват пакетов
4. Запустите процесс сопряжения с Mi Home
5. Во время pairing происходит обмен ключами
6. Найдите пакеты с характеристикой `Pairing` или `Key Exchange`
7. Извлеките bindkey из зашифрованного обмена

**Примечание**: Этот метод сложен и требует глубоких знаний BLE-протокола.

---

## Формат Bindkey

### Валидный формат
- **Длина**: 32 шестнадцатеричных символа (16 байт)
- **Пример**: `a1b2c3d4e5f6789012345678abcdef00`
- **Регистр**: неважен (принимается как нижний, так и верхний)

### Проверка валидности

```python
import re

def validate_bindkey(bindkey):
    """Проверка формата bindkey."""
    if not bindkey:
        return False
    
    # Удаление пробелов и тире
    bindkey = bindkey.replace(' ', '').replace('-', '').replace(':', '')
    
    # Проверка длины и hex-символов
    if len(bindkey) != 32:
        print(f"Invalid length: {len(bindkey)} (expected 32)")
        return False
    
    if not re.match(r'^[0-9A-Fa-f]{32}$', bindkey):
        print("Invalid characters (expected hex: 0-9, A-F)")
        return False
    
    return True

# Пример использования
bindkey = "a1b2c3d4e5f6789012345678abcdef00"
if validate_bindkey(bindkey):
    print("Bindkey is valid!")
```

---

## Настройка Add-On с Bindkey

### Обновление config.json (будущая версия)

После получения bindkey добавьте его в конфигурацию add-on:

```json
{
  "HCI_DEV": "hci0",
  "MISCALE_MAC": "D4:43:8A:CD:3F:76",
  "MISCALE_MODEL": "s800",
  "MISCALE_BINDKEY": "a1b2c3d4e5f6789012345678abcdef00",
  
  "MQTT_HOST": "127.0.0.1",
  "MQTT_PORT": 1883,
  "MQTT_USERNAME": "mqtt_xiaomi",
  "MQTT_PASSWORD": "your_password",
  "MQTT_PREFIX": "miscale",
  "MQTT_DISCOVERY": true,
  
  "DEBUG_LEVEL": "INFO",
  
  "USERS": [
    {
      "NAME": "User1",
      "SEX": "male",
      "GT": 60,
      "LT": 100,
      "HEIGHT": 175,
      "DOB": "1990-01-01"
    }
  ]
}
```

### Проверка работы с bindkey

После запуска add-on с настроенным bindkey проверьте логи:

```log
2025-11-18 10:00:00 - (INFO) Starting Xiaomi mi Scale v0.3.7...
2025-11-18 10:00:00 - (INFO) Loading Config From Options.json...
2025-11-18 10:00:00 - (DEBUG) MISCALE_MAC read from config: D4:43:8A:CD:3F:76
2025-11-18 10:00:00 - (DEBUG) MISCALE_MODEL read from config: s800
2025-11-18 10:00:00 - (DEBUG) MISCALE_BINDKEY configured: Yes (32 chars)
2025-11-18 10:00:05 - (DEBUG) S800 encrypted packet detected, decrypting...
2025-11-18 10:00:05 - (INFO) S800 data decrypted: weight=72.5 kg, impedance=520 Ω
2025-11-18 10:00:05 - (INFO) Publishing data to topic miscale/User1/weight
```

---

## Troubleshooting

### Проблема: Bindkey не извлекается через ADB backup

**Решение:**
- Убедитесь, что в Mi Home включён режим разработчика
- Попробуйте метод с облачным API
- Проверьте, что устройство добавлено в Mi Home и выполнено хотя бы одно измерение

### Проблема: python-miio не находит устройство

**Решение:**
- Убедитесь, что устройство привязано к вашей учётной записи Xiaomi
- Выберите правильный регион (cn, de, i2, ru, sg, us)
- Проверьте учётные данные

### Проблема: Расшифровка не работает с bindkey

**Решение:**
- Проверьте формат bindkey (32 hex символа)
- Убедитесь, что bindkey соответствует именно этому устройству (MAC)
- Попробуйте переполучить bindkey через другой метод
- Проверьте логи на ошибки расшифровки

### Проблема: Устройство не найдено в списке облака

**Решение:**
- Убедитесь, что S800 добавлен в Mi Home
- Попробуйте синхронизировать устройства (обновить список в Mi Home)
- Проверьте, что используется та же учётная запись

---

## Безопасность

### Важные рекомендации

⚠️ **Bindkey — это секретный ключ устройства**
- Не публикуйте bindkey в открытых источниках
- Не передавайте bindkey третьим лицам
- Храните bindkey в защищённом месте
- При публикации конфигураций заменяйте bindkey на `XXXXXXXX...`

### Ротация ключей

При необходимости обновить bindkey:
1. Удалите устройство из Mi Home
2. Выполните сброс к заводским настройкам (удерживайте кнопку на весах)
3. Заново добавьте устройство в Mi Home
4. Извлеките новый bindkey

---

## Дополнительные ресурсы

- [Mi Home Protocol Documentation](https://iot.mi.com/new/doc/home)
- [python-miio GitHub](https://github.com/rytilahti/python-miio)
- [Xiaomi Cloud Tokens Extractor](https://github.com/PiotrMachowski/Xiaomi-cloud-tokens-extractor)
- [Mi Beacon Protocol Specification](https://iot.mi.com/new/doc/embedded-development/ble/ble-mibeacon)
- [Home Assistant Xiaomi Integration](https://www.home-assistant.io/integrations/xiaomi_miio/)

---

## Контакты

При возникновении проблем с извлечением bindkey:
1. Проверьте раздел Troubleshooting
2. Обратитесь к сообществу Home Assistant
3. Создайте issue в репозитории проекта с описанием проблемы (без публикации bindkey!)

---

> **Следующий шаг**: После получения bindkey вернитесь к основной документации интеграции S800 для настройки add-on.
