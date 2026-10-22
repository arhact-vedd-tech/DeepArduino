## Глава 11: Проект «Метеостанция на столе»

### Цель

Собрать всё из Модуля III в один работающий проект. Главное — **уйти от `delay()`**: DHT требует паузы 2 секунды между опросами, но за эти 2 секунды плата должна продолжать работать (обновлять дисплей, читать Serial, мигать индикатором). Это первый проект, где `millis()` не учебный пример, а необходимость.

### Состав

| Компонент | Пин | Примечание |
|---|---|---|
| DHT11/DHT22 | D2 | Подтяжка 10 кОм (если модуль голый) |
| LCD 1602 I2C | A4 (SDA), A5 (SCL) | Адрес `0x27` или `0x3F` |
| Светодиод-индикатор | D8 | Мигает раз в секунду — «жива» |
| Serial | USB | Дубль данных для отладки |

Опционально: кнопка на D3 для принудительного обновления (разберём в задании).

### Логика

Три независимые задачи, каждая со своим таймером через `millis()`:

1. **DHT** — опрос раз в **2000 мс** (DHT11 не терпит чаще).
2. **LCD** — обновление раз в **500 мс** (чтобы не мерцало, но и не отставало).
3. **Индикатор** — мигание раз в **1000 мс**.

`loop()` крутится без задержек, каждая задача проверяет: «прошло ли моё время?».

### Код

```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>

// --- Пины и константы ---
#define DHT_PIN   2
#define DHT_TYPE  DHT11
#define LED_PIN   8

const unsigned long DHT_INTERVAL  = 2000;
const unsigned long LCD_INTERVAL  = 500;
const unsigned long LED_INTERVAL  = 1000;

// --- Объекты ---
DHT dht(DHT_PIN, DHT_TYPE);
LiquidCrystal_I2C lcd(0x27, 16, 2);

// --- Состояние ---
float temperature = NAN;
float humidity    = NAN;
bool  dhtValid    = false;

unsigned long lastDhtTime = 0;
unsigned long lastLcdTime = 0;
unsigned long lastLedTime = 0;
bool ledState = false;

void setup() {
  Serial.begin(9600);
  pinMode(LED_PIN, OUTPUT);
  dht.begin();
  lcd.init();
  lcd.backlight();
  lcd.print(F("Meteo start..."));
  Serial.println(F("=== Meteo start ==="));
}

void loop() {
  unsigned long now = millis();

  // --- Задача 1: DHT ---
  if (now - lastDhtTime >= DHT_INTERVAL) {
    lastDhtTime = now;
    float t = dht.readTemperature();
    float h = dht.readHumidity();
    if (isnan(t) || isnan(h)) {
      dhtValid = false;
      Serial.println(F("DHT read failed"));
    } else {
      temperature = t;
      humidity = h;
      dhtValid = true;
    }
  }

  // --- Задача 2: LCD ---
  if (now - lastLcdTime >= LCD_INTERVAL) {
    lastLcdTime = now;
    lcd.setCursor(0, 0);
    if (dhtValid) {
      lcd.print(F("T: "));
      lcd.print(temperature, 1);
      lcd.print(F(" C   "));
    } else {
      lcd.print(F("T: --.- C   "));
    }
    lcd.setCursor(0, 1);
    if (dhtValid) {
      lcd.print(F("H: "));
      lcd.print(humidity, 1);
      lcd.print(F(" %   "));
    } else {
      lcd.print(F("H: --.- %   "));
    }
  }

  // --- Задача 3: индикатор ---
  if (now - lastLedTime >= LED_INTERVAL) {
    lastLedTime = now;
    ledState = !ledState;
    digitalWrite(LED_PIN, ledState);
  }

  // --- Задача 4: Serial по запросу ---
  if (Serial.available() > 0) {
    char c = Serial.read();
    if (c == '?') {
      Serial.print(F("T="));
      Serial.print(temperature, 1);
      Serial.print(F(" C, H="));
      Serial.print(humidity, 1);
      Serial.println(F(" %"));
    }
  }
}
```

### Разбор

**Почему `millis()` а не `delay()`**: если поставить `delay(2000)` после чтения DHT, то LCD будет обновляться раз в 2 секунды, индикатор — дёргаться, Serial — не отвечать. `millis()` позволяет трём задачам идти параллельно в одном потоке.

**Паттерн «неблокирующий таймер»**:

```cpp
if (now - lastTime >= INTERVAL) {
  lastTime = now;
  // действие
}
```

Запомни его. Он встретится в главах 12, 13, 14, 15. Разность `unsigned long` корректна при переполнении `millis()` (глава 3) — это не случайность, а причина, почему пишут именно `now - lastTime`, а не `now > lastTime + INTERVAL`.

**Флаг `dhtValid`**: если датчик сбойнул, на экране `--.-`, а не последнее значение. Это честнее: пользователь видит, что данные устарели, а не думает, что температура не меняется.

**Serial-команда `?`**: запрос текущих значений. Показывает, что `loop()` не блокирован — ответ приходит в любой момент, а не «через 2 секунды после опроса DHT».

### Что может пойти не так

- **LCD показывает мусор** — не тот адрес I2C или плохой контакт. См. главу 10.
- **DHT всегда `--.-`** — нет подтяжки, не тот пин, или опрос чаще 1 Гц (здесь 2 с — нормально).
- **Индикатор мигает неравномерно** — если в какой-то ветке появился `delay`, таймеры сбиваются. Проверь, что нигде нет блокирующих вызовов.
- **`millis()` «застыл»** — не бывает. Но если `lastDhtTime` инициализирован не нулём, первое срабатывание может быть раньше/позже. Здесь `0` — корректно.
- **Serial не отвечает на `?`** — line ending в мониторе добавлен, а скетч читает один символ. Отправляй `?` без line ending, либо обрабатывай `\n` отдельно.

### Задание

1. Собери проект. Добейся, чтобы на LCD были T и H, индикатор мигал, а по `?` в Serial приходил ответ.
2. Добавь кнопку на D3 (`INPUT_PULLUP`) с антидребезгом из главы 5. По нажатию — **внеочередной** опрос DHT (сбрось `lastDhtTime` в `0`, чтобы условие сработало немедленно). Это демонстрация того, что таймер можно «подтолкнуть» извне.
3. Добавь на LCD третью строку... стоп, LCD 1602 — две строки. Вместо этого: выведи на вторую строку честный «возраст» данных — `(millis() - lastDhtTime) / 1000` секунд. Если DHT сбойнул и `lastDhtTime` не обновился, возраст растёт — видно, что данные старые.