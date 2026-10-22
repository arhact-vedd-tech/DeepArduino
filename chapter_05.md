## Глава 5: Цифровой вход

### Цель

Научиться читать состояние кнопки и других цифровых сигналов. Понять, почему кнопка «звенит» и как с этим бороться.

### Что значит «цифровой вход»

Пин в режиме `INPUT` читает напряжение и возвращает `HIGH` (близко к 5 В) или `LOW` (близко к 0 В). Всё, что между — неопределённость, и Arduino решит это произвольно.

**Проблема висящего пина**: если к пину ничего не подключено, он «ловит воздух» — наводки, статику, соседние пины. `digitalRead()` будет возвращать случайные значения. Поэтому пин **всегда должен быть куда-то подтянут** — либо к 5 В, либо к GND.

### Два способа подключить кнопку

**Способ 1: внешний резистор (pull-down)**

```
5V → кнопка → пин 2
пин 2 → резистор 10 кОм → GND
```

Кнопка нажата — на пине 5 В (`HIGH`). Отпущена — резистор тянет к GND (`LOW`).

**Способ 2: внутренняя подтяжка (pull-up)** — правильный по умолчанию

```
пин 2 → кнопка → GND
```

```cpp
pinMode(BUTTON_PIN, INPUT_PULLUP);
```

Внутри ATmega328P есть резистор ~20–50 кОм между пином и 5 В. Логика **инвертирована**:

| Кнопка | Пин |
|---|---|
| Отпущена | `HIGH` |
| Нажата | `LOW` |

Это экономит резистор и провода. **Используй `INPUT_PULLUP` везде, где возможно.**

### Код: кнопка управляет светодиодом

```cpp
const byte BUTTON_PIN = 2;
const byte LED_PIN = 8;

void setup() {
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  if (digitalRead(BUTTON_PIN) == LOW) {  // нажата
    digitalWrite(LED_PIN, HIGH);
  } else {
    digitalWrite(LED_PIN, LOW);
  }
}
```

Светодиод горит, пока кнопка нажата. Просто, но есть проблема.

### Дребезг

Механические контакты кнопки при нажатии и отпускании «звенят» — замыкаются и размыкаются несколько раз за 5–20 мс. `digitalRead()` в цикле поймает это как несколько нажатий.

**Симптом**: одно нажатие — светодиод мигает несколько раз, счётчик прибавляет 3–5 вместо 1.

**Решение 1: задержка** (просто, но блокирует)

```cpp
if (digitalRead(BUTTON_PIN) == LOW) {
  delay(20);  // ждём, пока дребезг утихнет
  if (digitalRead(BUTTON_PIN) == LOW) {  // всё ещё нажата — это не дребезг
    // действие
  }
}
```

Минус: `delay` блокирует `loop()`. Для одной кнопки — терпимо. Для нескольких — нет.

**Решение 2: `millis()`** (правильно, не блокирует)

```cpp
const byte BUTTON_PIN = 2;
const unsigned long DEBOUNCE_MS = 20;

bool lastStableState = HIGH;
bool lastReadState = HIGH;
unsigned long lastChangeTime = 0;

void setup() {
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  Serial.begin(9600);
}

void loop() {
  bool currentRead = digitalRead(BUTTON_PIN);

  if (currentRead != lastReadState) {
    lastChangeTime = millis();
    lastReadState = currentRead;
  }

  if ((millis() - lastChangeTime) > DEBOUNCE_MS) {
    if (currentRead != lastStableState) {
      lastStableState = currentRead;
      if (lastStableState == LOW) {
        Serial.println(F("Button pressed"));
      }
    }
  }
}
```

Логика: если состояние изменилось — запоминаем время. Если с тех пор прошло 20 мс и состояние всё ещё то же — считаем его стабильным.

### Конечный автомат: нажатие vs удержание

Если нужно различать «короткое нажатие» и «долгое удержание» — нужен автомат. Простой пример:

```cpp
enum ButtonState { IDLE, PRESSED, HELD };
ButtonState state = IDLE;
unsigned long pressTime = 0;
const unsigned long HOLD_MS = 1000;

void loop() {
  bool pressed = (digitalRead(BUTTON_PIN) == LOW);
  unsigned long now = millis();

  switch (state) {
    case IDLE:
      if (pressed) {
        state = PRESSED;
        pressTime = now;
      }
      break;
    case PRESSED:
      if (!pressed) {
        Serial.println(F("Short press"));
        state = IDLE;
      } else if (now - pressTime > HOLD_MS) {
        Serial.println(F("Long press"));
        state = HELD;
      }
      break;
    case HELD:
      if (!pressed) state = IDLE;
      break;
  }
}
```

Это шаблон, который пригодится в главах 12, 14, 15.

### Что может пойти не так

- **Кнопка «не работает»** — забыл `INPUT_PULLUP` или внешний резистор. Пин висит в воздухе, читает случайное.
- **Логика инвертирована** — с `INPUT_PULLUP` нажатие = `LOW`, не `HIGH`. Проверяй условие.
- **Одно нажатие = несколько срабатываний** — дребезг. Нужен антидребезг.
- **Кнопка подключена к D0/D1** — конфликт с Serial. Используй D2–D13.
- **`digitalRead` возвращает `HIGH`, хотя кнопка нажата** — либо нет подтяжки, либо кнопка подключена не к GND, а к 5V (тогда логика другая).

### Задание

1. Собери: кнопка между пином 2 и GND, `INPUT_PULLUP`. Светодиод на пине 8. Скетч: **каждое нажатие** переключает светодиод (нажал — горит, нажал ещё — погас). Используй антидребезг через `millis()`.
2. Добавь второй светодиод на пин 9. Логика: короткое нажатие — переключает первый, долгое (>1 сек) — переключает второй. Используй конечный автомат.
3. Объясни, почему `if (digitalRead(BUTTON_PIN) == HIGH)` с `INPUT_PULLUP` означает «кнопка отпущена». (Ответ: подтяжка тянет пин к 5 В, пока кнопка не замкнёт его на GND.)
