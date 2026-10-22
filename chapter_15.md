## Глава 15: Цифровые глаза на OLED (глаза + реакция на шум + часы)

### Цель

Собрать конечный автомат с тремя состояниями: **глаза** (моргание, зрачки, оглядывание) → **шум** (глаза прячутся) → **часы** (на `millis()`). Это самая сложная глава, потому что здесь три техники из предыдущих глав работают одновременно, и ни одна не блокирует другую.

### Состав

| Компонент | Пин | Примечание |
|---|---|---|
| OLED 128×64 SSD1306 I2C | A4 (SDA), A5 (SCL) | Адрес `0x3C`/`0x3D` |
| Аналоговый датчик шума (KY-038 и подобный) | A0 | Аналоговый выход, 0–1023 |
| Кнопка возврата | D2 | `INPUT_PULLUP`, опционально |

### Библиотеки

**FluxGarage RoboEyes** — специализированная библиотека для анимированных глаз на OLED. Умеет моргать (`setAutoblinker`), оглядываться (`setIdleMode`), менять настроение (`setMood`), закрывать/открывать глаза (`close`/`open`) . Работает поверх Adafruit SSD1306.

Установка: скачай `.zip` с GitHub FluxGarage/RoboEyes и установи через Library Manager в VS Arduino (или распакуй в `Documents/Arduino/libraries`). Зависимость — Adafruit SSD1306 + GFX.

**Важно**: RoboEyes закрывает/открывает глаза, но **не умеет** переключаться на часы или реагировать на шум. Это пишем сами поверх библиотеки.

### Архитектура: три состояния

```cpp
enum State { EYES, HIDING, CLOCK };
State currentState = EYES;

unsigned long stateTimer = 0;
const unsigned long HIDE_DURATION = 3000;   // прячется 3 секунды
const unsigned long CLOCK_DURATION = 10000; // часы 10 секунд
const int NOISE_THRESHOLD = 600;            // подбирается экспериментально
```

**Переходы**:
- `EYES` → `HIDING`: шум выше порога.
- `HIDING` → `CLOCK`: прошло `HIDE_DURATION`.
- `CLOCK` → `EYES`: прошло `CLOCK_DURATION` (или нажата кнопка).

### Шаг 1: Глаза (RoboEyes)

```cpp
#include <Adafruit_SSD1306.h>
#include <FluxGarage_RoboEyes.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);
RoboEyes<Adafruit_SSD1306> roboEyes(display);

void setupEyes() {
  roboEyes.begin(SCREEN_WIDTH, SCREEN_HEIGHT, 100);
  roboEyes.setAutoblinker(ON, 3, 2);  // моргает каждые 3±2 сек
  roboEyes.setIdleMode(ON, 2, 2);     // оглядывается каждые 2±2 сек
  roboEyes.setMood(DEFAULT);
  roboEyes.open();
}
```

`setAutoblinker(ON, 3, 2)` — моргание с интервалом 3 секунды ±2 случайных. `setIdleMode` — случайные взгляды в стороны . Оба работают автоматически, пока вызывается `roboEyes.update()` в `loop()`.

### Шаг 2: Шум

KY-038 имеет **два выхода**: цифровой (D0 — порог, HIGH/LOW) и аналоговый (A0 — сырое значение 0–1023) . Используем аналоговый: цифровой слишком грубый, и порог на потенциометре модуля не поддаётся точной настройке из кода.

```cpp
const byte NOISE_PIN = A0;

int readNoise() {
  // Усреднение 8 сэмплов — сглаживание случайных всплесков
  long sum = 0;
  for (byte i = 0; i < 8; i++) {
    sum += analogRead(NOISE_PIN);
    delay(1);
  }
  return sum / 8;
}
```

`NOISE_THRESHOLD` подбирается экспериментально: открой Serial, посмотри значения в тишине и при хлопке. Порог — где-то посередине, но ближе к тишине (чтобы не срабатывало на скрип стула).

### Шаг 3: Часы на `millis()`

Без RTC точность `millis()` уходит за сутки, но для демонстрации — достаточно. Счётчик секунд, минут, часов:

```cpp
int hours = 12, minutes = 0, seconds = 0;
unsigned long lastSecondTick = 0;

void updateClock() {
  unsigned long now = millis();
  if (now - lastSecondTick >= 1000) {
    lastSecondTick = now;
    seconds++;
    if (seconds >= 60) { seconds = 0; minutes++; }
    if (minutes >= 60) { minutes = 0; hours++; }
    if (hours >= 24) hours = 0;
  }
}

void drawClock() {
  display.clearDisplay();
  display.setTextSize(2);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(20, 20);
  // Формат ЧЧ:ММ:СС
  if (hours < 10) display.print('0');
  display.print(hours); display.print(':');
  if (minutes < 10) display.print('0');
  display.print(minutes); display.print(':');
  if (seconds < 10) display.print('0');
  display.print(seconds);
  display.display();
}
```

`display.display()` вызывается **только при обновлении секунды**, не каждый кадр — иначе мерцание и лишняя нагрузка на I2C.

### Шаг 4: Конечный автомат

```cpp
void loop() {
  unsigned long now = millis();

  switch (currentState) {

    case EYES:
      roboEyes.update();  // библиотека сама моргает и оглядывается
      if (readNoise() > NOISE_THRESHOLD) {
        roboEyes.close();           // прячем глаза
        currentState = HIDING;
        stateTimer = now;
      }
      break;

    case HIDING:
      // глаза закрыты, ничего не рисуем
      if (now - stateTimer >= HIDE_DURATION) {
        currentState = CLOCK;
        stateTimer = now;
        lastSecondTick = now;  // синхронизируем часы
      }
      break;

    case CLOCK:
      updateClock();
      drawClock();
      if (now - stateTimer >= CLOCK_DURATION) {
        currentState = EYES;
        roboEyes.open();
        roboEyes.setAutoblinker(ON, 3, 2);  // перезапускаем анимации
        roboEyes.setIdleMode(ON, 2, 2);
      }
      break;
  }
}
```

**Ключевой момент**: `roboEyes.update()` вызывается **только в состоянии `EYES`**. В `HIDING` и `CLOCK` библиотека не активна — мы управляем дисплеем напрямую. При возврате в `EYES` вызываем `open()` и заново включаем авто-анимации, потому что `close()` их сбрасывает.

### Что может пойти не так

- **`SSD1306 allocation failed`** — адрес не `0x3C`/`0x3D`. I2C-сканер в помощь (глава 10).
- **RoboEyes не компилируется** — библиотека не установлена или установлена не та версия. Нужна именно FluxGarage/RoboEyes, не путать с форками .
- **Шум срабатывает постоянно** — порог слишком низкий. Подними `NOISE_THRESHOLD`. Или датчик ловит вентилятор/компьютер — отодвинь.
- **Часы отстают/спешат** — `millis()` неточен по определению. Для демо — нормально. Если нужна точность — DS3231 по I2C (другая глава).
- **При переходе в `CLOCK` экран в мусоре** — `drawClock()` не вызывает `clearDisplay()` перед первой отрисовкой? Вызывай `display.clearDisplay()` в начале `drawClock()` (в коде выше он есть).
- **SRAM близко к пределу** — RoboEyes + Adafruit SSD1306 занимают буфер 1 КБ. Если добавишь ещё что-то тяжёлое — компилятор покажет. Следи, чтобы глобальные переменные не превышали ~1500 байт .

### Задание

1. Собери всё. Добейся, чтобы глаза моргали и оглядывались. Хлопни — прячутся. Через 3 секунды — часы. Через 10 — снова глаза.
2. Добавь кнопку на D2: в состоянии `CLOCK` нажатие сразу возвращает к глазам (не ждать 10 секунд).
3. Замени `millis()`-часы на «время с момента включения» в формате `MM:SS` (минуты:секунды). Проще, точнее, честнее.
4. **Опционально**: если нужна реальная точность — подключи DS3231 по I2C (адрес `0x68`) и замени `updateClock()` на чтение времени с модуля. Это выходит за рамки главы, но направление правильное.
