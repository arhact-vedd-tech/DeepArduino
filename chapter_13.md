## Глава 13: Универсальный ИК-контроллер с датчиком температуры

### Цель

Собрать устройство, которое принимает ИК-сигналы с любого пульта, позволяет назначить код кнопки на действие, и по запросу показывает температуру с DHT. Ключевая проблема, которую решаем: **коды кнопок у всех пультов разные**, поэтому сначала захват, потом маппинг.

### Состав

| Компонент | Пин | Примечание |
|---|---|---|
| ИК-приёмник (TSOP1838 / VS1838B) | D2 | Выход — цифровой, 38 кГц |
| ИК-передатчик (опционально) | D3 | Через резистор 100–220 Ом |
| DHT11/DHT22 | D4 | Подтяжка 10 кОм, если модуль голый |
| LCD 1602 I2C (или OLED) | A4/A5 | Адрес `0x27`/`0x3F` |
| Кнопка «режим» (опционально) | D5 | `INPUT_PULLUP` |

### Установка библиотеки

Нужна **IRremote** (Arduino-IRremote by Armin Joachimsmeyer). В VS Arduino: `Ctrl+Shift+P` → **VS Arduino: Library Manager** → поиск `IRremote` → установить.

**Важно**: современная IRremote (v3+) имеет другой API, чем старые учебники (v2.x). В примерах ниже — API v3+.

### Шаг 1: Захват кодов

Первый скетч — только приём и вывод в Serial. Задача: узнать, какой код выдаёт каждая кнопка твоего пульта.

```cpp
#include <IRremote.h>

const byte IR_RECEIVE_PIN = 2;

void setup() {
  Serial.begin(115200);
  IrReceiver.begin(IR_RECEIVE_PIN, ENABLE_LED_FEEDBACK);
  Serial.println(F("=== IR capture ready ==="));
}

void loop() {
  if (IrReceiver.decode()) {
    Serial.print(F("Protocol: "));
    Serial.print(getProtocolString(IrReceiver.decodedIRData.protocol));
    Serial.print(F("  Code: 0x"));
    Serial.println(IrReceiver.decodedIRData.command, HEX);

    IrReceiver.resume();
  }
}
```

**Что делать**: открой монитор (115200), нажми каждую кнопку пульта, запиши коды. Например:

```
Protocol: NEC  Code: 0x45
Protocol: NEC  Code: 0x46
Protocol: NEC  Code: 0x47
```

Эти значения — твой «словарь». Они уникальны для твоего пульта. У другого пульта будут другие.

**Если коды не появляются**: проверь питание приёмника (5V/GND), пин данных (D2), и что пульт направлен в «морду» приёмника. Некоторые приёмники требуют конденсатор 100 нФ между VCC и GND.

### Шаг 2: Маппинг и действия

Второй скетч — с назначенными действиями и DHT.

```cpp
#include <IRremote.h>
#include <DHT.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// --- Пины ---
const byte IR_RECEIVE_PIN = 2;
const byte DHT_PIN = 4;
#define DHT_TYPE DHT11

// --- Твои коды (замени на захваченные) ---
const uint16_t CODE_POWER   = 0x45;
const uint16_t CODE_TEMP    = 0x46;
const uint16_t CODE_LED     = 0x47;

// --- Объекты ---
DHT dht(DHT_PIN, DHT_TYPE);
LiquidCrystal_I2C lcd(0x27, 16, 2);

// --- Состояние ---
bool ledState = false;
const byte LED_PIN = 8;

void showTemperature() {
  float t = dht.readTemperature();
  float h = dht.readHumidity();

  lcd.clear();
  if (isnan(t) || isnan(h)) {
    lcd.print(F("DHT error"));
  } else {
    lcd.setCursor(0, 0);
    lcd.print(F("T: "));
    lcd.print(t, 1);
    lcd.print(F(" C"));
    lcd.setCursor(0, 1);
    lcd.print(F("H: "));
    lcd.print(h, 1);
    lcd.print(F(" %"));
  }
}

void setup() {
  Serial.begin(115200);
  dht.begin();
  lcd.init();
  lcd.backlight();
  lcd.print(F("IR ready"));
  pinMode(LED_PIN, OUTPUT);
  IrReceiver.begin(IR_RECEIVE_PIN, ENABLE_LED_FEEDBACK);
}

void loop() {
  if (IrReceiver.decode()) {
    uint16_t cmd = IrReceiver.decodedIRData.command;

    Serial.print(F("Got: 0x"));
    Serial.println(cmd, HEX);

    if (cmd == CODE_POWER) {
      ledState = !ledState;
      digitalWrite(LED_PIN, ledState);
      lcd.clear();
      lcd.print(ledState ? F("LED ON") : F("LED OFF"));
    }
    else if (cmd == CODE_TEMP) {
      showTemperature();
    }
    else if (cmd == CODE_LED) {
      lcd.clear();
      lcd.print(F("Hello"));
    }

    IrReceiver.resume();
  }
}
```

### Разбор

**`IrReceiver.decodedIRData.command`** — код кнопки (8 бит для NEC). `protocol` — тип протокола (NEC, Sony, RC5...). Для большинства бытовых пультов — NEC.

**Маппинг через константы** — правильный подход. Коды вынесены в `const uint16_t` вверху файла. При смене пульта меняешь только эти строки.

**DHT вызывается только по запросу** (кнопка `CODE_TEMP`), а не в `loop()` постоянно. Это решает проблему «DHT медленный»: опрос происходит редко, когда нужно, и не блокирует приём ИК.

**`IrReceiver.resume()`** — обязателен после каждого декодирования. Без него приёмник не примет следующий сигнал.

### Что может пойти не так

- **Коды не совпадают при повторном нажатии** — некоторые пульты шлют разные коды для «нажал» и «удерживаю». `decodedIRData.command` при удержании может быть `0x00` или повторяться. Для этой главы достаточно однократных нажатий.
- **`getProtocolString` не компилируется** — в некоторых версиях IRremote функция называется иначе. Проверь примеры из библиотеки (`File → Examples → IRremote`).
- **DHT показывает `NaN`** — см. главу 9.
- **LCD пустой** — адрес I2C или контраст (глава 10).
- **ИК-приёмник ловит помехи от LCD/подсветки** — некоторые LCD-модули излучают на 38 кГц. Попробуй отодвинуть приёмник или использовать экран.

### Задание

1. Собери скетч захвата. Запиши коды 5 кнопок своего пульта. Сравни с таблицей, если она есть в инструкции к пульту (редко, но бывает).
2. Собери второй скетч. Назначь: кнопка 1 — LED ON/OFF, кнопка 2 — показать температуру, кнопка 3 — очистить экран.
3. Добавь кнопку на D5 (физическую, `INPUT_PULLUP`): при нажатии — режим «обучение». В этом режиме следующее нажатие на пульте **перезаписывает** код для выбранной кнопки и сохраняет в EEPROM (библиотека `EEPROM.h`). После перезагрузки код остаётся. Это полноценный «обучаемый пульт» — то, что делают коммерческие универсальные пульты .
