# Analysis of Nekogram decompiled source code - phone exfiltration backdoor discovered
#### Анализ декомпилированного исходного кода Nekogram — обнаружена бєкдор для сбора номеров телефонов

![code](code.jpg)

## EN

### Summary
A phone number-stealing backdoor has been identified within the Nekogram Android client. The investigation reveals that the application contains obfuscated logic designed to silently collect and upload the phone numbers of all accounts logged into the app. This malicious behavior is present in distributed versions, including the version available on the **Google Play Store**.

### The Backdoor: `Extra.java`
The malicious code is concealed within `Extra.java`. While the [public GitHub repository](https://github.com/Nekogram/Nekogram/blob/6a30d8f540142fd4f862a505ba2e1cf5f53ea1a2/TMessagesProj/src/main/java/tw/nekomimi/nekogram/Extra.java.example) displays a "clean" example version of this file, the compiled binaries used by the public contain the exfiltration logic.

### Analysis

#### 1. Data Collection & Account Linkage
The core logic (found in function `uo5.g()`, restored as `logNumberPhones`) iterates through every Telegram account active in the app (up to 8 accounts).
```java
for (int i2 = 0; i2 < 8; i2++) {
    UserConfig userConfig = UserConfig.getInstance(i2);
    if (userConfig.A() && (currentUser = userConfig.o()) != null && (phoneNumber = currentUser.f) != null) {
        mapWithAccounts.put(String.valueOf(currentUser.a), phoneNumber);
    }
}
```
**Impact:** This allows the developer to link multiple accounts to a single physical user/device, destroying the anonymity of users who use different accounts for different purposes.

#### 2. Silent Exfiltration via Inline Queries
The data is exfiltrated using **Inline Queries**. This is a highly stealthy method because it does not leave a trace in the user's "Sent Messages" or chat history.
- **Target Bot:** `@nekonotificationbot` (ID `1190800416`)
- **Method:** `InlineBotHelper.Query(...)`
- **Payload:** A JSON map of `UserID -> Phone Number` prefixed with a secret key.
- **Secret Key:** `741ad28818eab17668bc2c70bd419fc25ff56481758a4ac87e7ca164fb6ae1b1`

The final exfiltrated string looks like:
`741ad2...ae1b1{"123456789": "+79001234567", ...}`

#### 3. Decrypted Constants
Using a custom decryption tool, we recovered hidden strings used by the backdoor:

| ID | Decrypted Value | Context |
|----|-----------------|---------|
| -7227028090432512756 | `873ffaceba76e791ff2491224a3cdb49` | APP_HASH |
| -7227026067502916340 | **`nekonotificationbot`** | Primary Exfiltration Target |
| -7227026153402262260 | **`tgdb_search_bot`** | OSINT Bot Reference |
| -7227025642301154036 | **`usinfobot`** | OSINT Bot Reference |
| -7227027729655259892 | `741ad2...ae1b1` | Shared Secret / Hash |

Also APP_ID was found: `442495`

### Conclusion
Nekogram is actively harvesting private user data. By sending phone numbers and User IDs to a centralized bot, the developers can build a database mapping Telegram users to real-world identities. The use of obfuscation and inline queries demonstrates a clear intent to hide this behavior from both users and security researchers.

---

## RU

### Обзор
В Android-клиенте Nekogram обнаружен бэкдор, ворующий номера телефонов. Расследование показало, что приложение содержит обфусцированный код, предназначенный для скрытого сбора и выгрузки номеров телефонов всех аккаунтов, вошедших в приложение. Это вредоносное поведение присутствует в публичных сборках, включая версию из **Google Play**.

### Бэкдор в `Extra.java`
Вредоносный код спрятан в файле `Extra.java`. Примечательно, что в [публичном репозитории GitHub](https://github.com/Nekogram/Nekogram/blob/6a30d8f540142fd4f862a505ba2e1cf5f53ea1a2/TMessagesProj/src/main/java/tw/nekomimi/nekogram/Extra.java.example) этот файл представлен в виде «чистого» примера, однако скомпилированные бинарные файлы, распространяемые среди пользователей, содержат логику кражи данных.

### Анализ

#### 1. Сбор данных и связывание аккаунтов
Основная логика (функция `uo5.g()`, восстановленное название: `logNumberPhones`) перебирает все учетные записи Telegram, активные в приложении (до 8 штук).
```java
for (int i2 = 0; i2 < 8; i2++) {
    UserConfig userConfig = UserConfig.getInstance(i2);
    if (userConfig.A() && (currentUser = userConfig.o()) != null && (phoneNumber = currentUser.f) != null) {
        mapWithAccounts.put(String.valueOf(currentUser.a), phoneNumber);
    }
}
```
**Последствия:** Это позволяет разработчику связать несколько аккаунтов с одним физическим пользователем/устройством, уничтожая анонимность пользователей, использующих разные аккаунты для разных целей.

#### 2. Скрытый слив через Inline-запросы
Данные передаются с помощью **Inline-запросов** (инлайн-запросов). Это крайне скрытный метод, так как он не оставляет следов в истории чатов или в папке «Отправленные».
- **Целевой бот:** `@nekonotificationbot` (ID `1190800416`)
- **Метод:** `InlineBotHelper.Query(...)`
- **Полезная нагрузка:** JSON-карта `UserID -> Номер телефона`, дополненная секретным ключом.
- **Секретный ключ:** `741ad28818eab17668bc2c70bd419fc25ff56481758a4ac87e7ca164fb6ae1b1`

Итоговая строка с похищенными данными выглядит так:
`741ad2...ae1b1{"123456789": "+79001234567", ...}`

#### 3. Расшифрованные константы
С помощью кастомного инструмента дешифровки были восстановлены скрытые строки, используемые бэкдором:

| ID | Расшифрованное значение | Контекст |
|----|-------------------------|----------|
| -7227028090432512756 | `873ffaceba76e791ff2491224a3cdb49` | APP_HASH |
| -7227026067502916340 | **`nekonotificationbot`** | Основной бот для слива |
| -7227026153402262260 | **`tgdb_search_bot`** | Упоминание OSINT-бота |
| -7227025642301154036 | **`usinfobot`** | Упоминание OSINT-бота |
| -7227027729655259892 | `741ad2...ae1b1` | Общий секрет / Хеш |

Также был найден APP_ID: `442495`

### Заключение
Nekogram активно собирает приватные данные пользователей. Отправляя номера телефонов и User ID на централизованный бот, разработчики могут создавать базу данных, связывающую пользователей Telegram с их реальными личностями. Использование обфускации и инлайн-запросов доказывает явное намерение скрыть эту деятельность как от пользователей, так и от исследователей безопасности.