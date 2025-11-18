# Тест расшифровки данных S800 с реальным bindkey

## Информация об устройстве
- **MAC**: `D4:43:8A:CD:3F:76`
- **Bindkey**: `4a4c63084637be44309909a88f7e6dc3`
- **Модель**: `xiaomi.scales.ms116`

## Примеры пакетов для расшифровки

### Пакет 1: Frame Counter 0x0A
```
Service Data: 4859e2510a9ec3045df18aefde c883fda30000 00bac0767f
Timestamp: 08:43:05

Разбор:
48          - Frame Control (encrypted)
59 e2       - Product ID (part)
51          - (unknown)
e2          - (unknown - возможно, часть Product ID)
0a          - Frame Counter
9e c3 04 5d f1 8a - Nonce data
ef de c8 83 fd a3 00 00 00 ba c0 76 7f - Ciphertext + Tag (13 байт)
```

### Пакет 2: Frame Counter 0x0B
```
Service Data: 4859e2510bcece0cd03325b8d9 2da8760300 0000df81da0a
Timestamp: 08:43:07
```

### Пакет 3: Frame Counter 0x0C
```
Service Data: 4859e2510c7dc5cf47f133ba10 98af309300 0000t09ceU55f7
Timestamp: 08:43:08
```

### Открытый пакет (для сравнения)
```
Service Data: 5859e25109763fcd8a43d4443d88570000000332 5d30
Timestamp: 08:43:00

58          - Frame Control (открытый)
59 e2 51 09 - Product ID + Frame Counter
76 3f cd 8a 43 d4 - MAC (reversed)
44 3d       - Object ID
88 57 00 00 00 03 - Data
32 5d 30    - Additional data
```

## Структура Mi Beacon для расшифровки

### Формат nonce (12 байт)
Согласно Mi Beacon спецификации:
```
Nonce = MAC (6 байт) + Product ID (2 байта) + Frame Counter (4 байта)
```

Для устройства S800:
```
MAC: D4:43:8A:CD:3F:76 (в прямом порядке для nonce)
Product ID: Требуется определить из пакетов
Frame Counter: 0x0000000A (для первого пакета, little-endian 32-bit)
```

### Проблема: Определение Product ID
Из пакетов видно: `59 e2 51 e2`
- Возможные варианты:
  - Product ID = 0xE259 (little-endian)
  - Product ID = 0x59E2 (big-endian)
  - Или это часть другой структуры

Требуется экспериментальная проверка обоих вариантов.

## Python-скрипт для расшифровки

```python
#!/usr/bin/env python3
from Crypto.Cipher import AES
import binascii

def decrypt_mibeacon(ciphertext_hex, bindkey_hex, mac_hex, product_id_hex, frame_counter):
    """
    Расшифровка Mi Beacon пакета AES-CCM.
    
    Args:
        ciphertext_hex: Зашифрованные данные (hex string)
        bindkey_hex: Ключ шифрования (32 hex chars)
        mac_hex: MAC-адрес устройства (12 hex chars)
        product_id_hex: Product ID (4 hex chars)
        frame_counter: Frame counter (int)
    """
    # Конвертация hex в bytes
    bindkey = bytes.fromhex(bindkey_hex)
    mac = bytes.fromhex(mac_hex)
    product_id = bytes.fromhex(product_id_hex)
    ciphertext = bytes.fromhex(ciphertext_hex)
    
    # Формирование nonce (12 байт)
    # Frame counter в little-endian 32-bit
    frame_counter_bytes = frame_counter.to_bytes(4, byteorder='little')
    nonce = mac + product_id + frame_counter_bytes
    
    print(f"Bindkey: {bindkey.hex()}")
    print(f"MAC: {mac.hex()}")
    print(f"Product ID: {product_id.hex()}")
    print(f"Frame Counter: {frame_counter} (0x{frame_counter:08x})")
    print(f"Nonce: {nonce.hex()}")
    print(f"Ciphertext length: {len(ciphertext)} bytes")
    
    # Разделение ciphertext и tag (последние 4 байта)
    tag_length = 4
    encrypted_payload = ciphertext[:-tag_length]
    tag = ciphertext[-tag_length:]
    
    print(f"Encrypted payload: {encrypted_payload.hex()}")
    print(f"Tag: {tag.hex()}")
    
    try:
        # Создание cipher AES-CCM
        cipher = AES.new(bindkey, AES.MODE_CCM, nonce=nonce, mac_len=tag_length)
        
        # Расшифровка и верификация
        plaintext = cipher.decrypt_and_verify(encrypted_payload, tag)
        
        print(f"\n✅ Decryption successful!")
        print(f"Plaintext: {plaintext.hex()}")
        print(f"Plaintext (bytes): {plaintext}")
        print(f"Length: {len(plaintext)} bytes")
        
        return plaintext
        
    except ValueError as e:
        print(f"\n❌ Decryption failed: {e}")
        return None

# Тест с реальными данными
if __name__ == "__main__":
    # Данные устройства
    BINDKEY = "4a4c63084637be44309909a88f7e6dc3"
    MAC = "d4438acd3f76"
    
    # Пакет 1: Frame Counter 0x0A
    print("=" * 80)
    print("Пакет 1: Frame Counter 0x0A")
    print("=" * 80)
    
    # Попытка 1: Product ID = 0xE259 (LE)
    CIPHERTEXT_1 = "efde c883fda30000 00bac0767f".replace(" ", "")
    PRODUCT_ID_1 = "59e2"  # Little-endian
    FRAME_COUNTER_1 = 0x0A
    
    print("\n--- Попытка с Product ID = 0xE259 (LE) ---")
    result = decrypt_mibeacon(CIPHERTEXT_1, BINDKEY, MAC, PRODUCT_ID_1, FRAME_COUNTER_1)
    
    if not result:
        # Попытка 2: Product ID = 0xE259 (BE)
        print("\n--- Попытка с Product ID = 0x59E2 (BE) ---")
        PRODUCT_ID_2 = "e259"  # Big-endian
        result = decrypt_mibeacon(CIPHERTEXT_1, BINDKEY, MAC, PRODUCT_ID_2, FRAME_COUNTER_1)
    
    if not result:
        # Попытка 3: Использовать все байты 59 e2 51
        print("\n--- Попытка с расширенным Product ID ---")
        # Возможно, nonce формируется иначе
        pass
    
    # Пакет 2: Frame Counter 0x0B
    print("\n" + "=" * 80)
    print("Пакет 2: Frame Counter 0x0B")
    print("=" * 80)
    CIPHERTEXT_2 = "cece0cd03325b8d9 2da8760300 0000df81da0a".replace(" ", "")
    FRAME_COUNTER_2 = 0x0B
    
    print("\n--- Попытка с Product ID = 0xE259 (LE) ---")
    result = decrypt_mibeacon(CIPHERTEXT_2, BINDKEY, MAC, PRODUCT_ID_1, FRAME_COUNTER_2)
```

## Ожидаемый результат

После успешной расшифровки мы должны увидеть:
- **Object ID** (2 байта): тип данных (вес, импеданс)
- **Данные измерения**: вес в граммах или единицах * 100
- **Дополнительные параметры**: импеданс, единицы измерения

## Следующие шаги

1. ✅ Получен bindkey
2. ⏳ Определить правильный Product ID
3. ⏳ Расшифровать пакеты
4. ⏳ Определить Object ID для веса и импеданса
5. ⏳ Создать парсер для расшифрованных данных
6. ⏳ Интегрировать в основной код
