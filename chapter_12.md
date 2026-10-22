## Глава 12: Управление по Serial

### Цель

Сделать так, чтобы платой можно было управлять командами из монитора порта. Не «прочитать символ — включить светодиод», а нормальный разбор строк, несколько команд, конечный автомат состояний. Это закрепление главы 7 и подготовка к главе 13 (ИК-контроллер).

### Что будем делать

Прошивка с меню:

| Команда | Действие |
|---|---|
| `LED ON` | светодиод на D8 горит |
| `LED OFF` | светодиод на D8 погас |
| `PWM 128` | ШИМ на D9 = 128 (0–255) |
| `SERVO 90` | серво на D10 = 90° |
| `STATUS` | вывести текущее состояние |
| `HELP` | список команд |

Плюс **конечный автомат**: плата помнит, в каком состоянии находится (LED вкл/выкл, PWM значение, угол серво).

### Почему не `Serial.readString()`

`readString()` читает в `String`, ждёт таймаут, фрагментирует SRAM. На Uno с 2 КБ это путь к зависанию через час работы. Правильно — **читать посимвольно в `char[]` до `\n`**, потом разбирать.

### Код: чтение строки

```cpp
const byte CMD_MAX = 32;
char cmdBuf[CMD_MAX];
byte cmdLen = 0;

bool readCommand() {
  while (Serial.available() > 0) {
    char c = Serial.read();
    if (c == '\n' || c == '\r') {
      if (cmdLen > 0) {
        cmdBuf[cmdLen] = '\0';  // завершаем C-строку
        cmdLen = 0;
        return true;
      }
    } else if (cmdLen < CMD_MAX - 1) {
      cmdBuf[cmdLen++] = c;
    }
  }
  return false;
}
```

**Что здесь важно**:
- `cmdBuf[CMD_MAX]` — фиксированный буфер, никакого `String`.
- `cmdLen < CMD_MAX - 1` — защита от переполнения (глава 3: выход за границы массива не проверяется).
- `'\0'` — завершитель C-строки. Без него `strcmp` и `strtok` будут читать мусор.
- Возврат `true`, когда строка собрана. `false` — данных ещё нет.

### Код: разбор и диспетчеризация

```cpp
#include <Servo.h>

const byte LED_PIN   = 8;
const byte PWM_PIN   = 9;
const byte SERVO_PIN = 10;

Servo myservo;

// --- Состояние ---
bool ledState = false;
byte pwmValue = 0;
byte servoAngle = 90;

void printHelp() {
  Serial.println(F("Commands:"));
  Serial.println(F("  LED ON | LED OFF"));
  Serial.println(F("  PWM 0..255"));
  Serial.println(F("  SERVO 0..180"));
  Serial.println(F("  STATUS"));
  Serial.println(F("  HELP"));
}

void printStatus() {
  Serial.print(F("LED="));
  Serial.print(ledState ? F("ON") : F("OFF"));
  Serial.print(F("  PWM="));
  Serial.print(pwmValue);
  Serial.print(F("  SERVO="));
  Serial.println(servoAngle);
}

void handleCommand(char* cmd) {
  // Первое слово — команда
  char* verb = strtok(cmd, " ");
  if (verb == NULL) return;

  if (strcmp(verb, "LED") == 0) {
    char* arg = strtok(NULL, " ");
    if (arg == NULL) { Serial.println(F("ERR: LED needs ON/OFF")); return; }
    if (strcmp(arg, "ON") == 0) {
      ledState = true;
      digitalWrite(LED_PIN, HIGH);
      Serial.println(F("OK LED ON"));
    } else if (strcmp(arg, "OFF") == 0) {
      ledState = false;
      digitalWrite(LED_PIN, LOW);
      Serial.println(F("OK LED OFF"));
    } else {
      Serial.println(F("ERR: LED arg must be ON or OFF"));
    }

  } else if (strcmp(verb, "PWM") == 0) {
    char* arg = strtok(NULL, " ");
    if (arg == NULL) { Serial.println(F("ERR: PWM needs number")); return; }
    int v = atoi(arg);
    if (v < 0 || v > 255) { Serial.println(F("ERR: PWM range 0..255")); return; }
    pwmValue = (byte)v;
    analogWrite(PWM_PIN, pwmValue);
    Serial.print(F("OK PWM "));
    Serial.println(pwmValue);

  } else if (strcmp(verb, "SERVO") == 0) {
    char* arg = strtok(NULL, " ");
    if (arg == NULL) { Serial.println(F("ERR: SERVO needs number")); return; }
    int v = atoi(arg);
    if (v < 0 || v > 180) { Serial.println(F("ERR: SERVO range 0..180")); return; }
    servoAngle = (byte)v;
    myservo.write(servoAngle);
    Serial.print(F("OK SERVO "));
    Serial.println(servoAngle);

  } else if (strcmp(verb, "STATUS") == 0) {
    printStatus();

  } else if (strcmp(verb, "HELP") == 0) {
    printHelp();

  } else {
    Serial.print(F("ERR: unknown command '"));
    Serial.print(verb);
    Serial.println(F("'"));
  }
}
```

### `setup` и `loop`

```cpp
void setup() {
  Serial.begin(9600);
  pinMode(LED_PIN, OUTPUT);
  pinMode(PWM_PIN, OUTPUT);
  myservo.attach(SERVO_PIN);
  myservo.write(servoAngle);
  Serial.println(F("=== Serial control ready ==="));
  printHelp();
}

void loop() {
  if (readCommand()) {
    handleCommand(cmdBuf);
  }
}
```

### Разбор

**`strtok`** — разбивает строку по разделителю, возвращает первое слово при первом вызове, затем `NULL`-аргумент — следующее. Модифицирует строку на месте (вставляет `\0`), поэтому `cmdBuf` должен быть изменяемым — `char[]`, не `const char*`.

**`atoi`** — строка → целое. Возвращает `0` при ошибке, поэтому проверяй диапазон **после** преобразования. Если пользователь ввёл `PWM abc`, `atoi` вернёт `0` — команда выполнится с `PWM 0`. Это баг, если не отличать «0» от «не число». Правильнее — `strtol` с проверкой конца строки, но для учебного проекта `atoi` + диапазон достаточно. В задании — исправить.

**Конечный автомат**: состояние (`ledState`, `pwmValue`, `servoAngle`) хранится в глобальных переменных. Команда меняет состояние и **сразу применяет** его к железу. `STATUS` читает состояние, не трогая железо. Это разделение «состояние / действие» — основа любой прошивки с меню (главы 13–15).

**Почему `handleCommand` принимает `char*`, а не `const char*`** — `strtok` модифицирует. Если передать `const`, компилятор поругается (и будет прав: `strtok` пишет в строку).

### Монитор порта: настройка

В VS Arduino:
- **Baud rate**: 9600 (как в `Serial.begin`).
- **Line ending**: **Newline** (`\n`). Скетч ждёт `\n` или `\r` как конец команды. Если стоит «No line ending» — команда не завершится, `readCommand()` не вернёт `true` .

### Что может пойти не так

- **Команда не выполняется** — line ending не Newline. Скетч ждёт `\n`.
- **`ERR: unknown command` на правильную команду** — лишние пробелы в начале (` LED ON`). `strtok` с разделителем `" "` пропустит ведущие пробелы, но если ввести `LED  ON` (два пробела) — второй `strtok(NULL, " ")` вернёт `ON`, всё ок. Проблема — в невидимых символах (`\r` от Windows-монитора). Скетч обрабатывает `\r` как конец строки, так что должно работать.
- **`PWM abc` ставит 0** — `atoi` не отличает ошибку от нуля. См. задание 3.
- **Серво дёргается при каждом `SERVO`** — нормально, это позиционирование.
- **Буфер переполнен** — если ввести строку длиннее 31 символа, лишние символы отбрасываются (проверка `cmdLen < CMD_MAX - 1`). Команда будет обрезана, скорее всего — `ERR: unknown`.

### Задание

1. Собери проект. Проверь все команды. Убедись, что `STATUS` показывает то, что ты установил.
2. Добавь команду `BLINK N` — мигает светодиодом N раз с интервалом 200 мс. **Не через `delay`** — через счётчик в `loop()` и `millis()`. Это закрепление главы 11.
3. Замени `atoi` на `strtol` с проверкой: если после числа остались символы — `ERR: not a number`. Подсказка: `char* end; long v = strtol(arg, &end, 10); if (*end != '\0') { /* ошибка */ }`.
4. Добавь команду `RESET` — сбрасывает всё состояние в начальное (LED OFF, PWM 0, SERVO 90) и применяет к железу.
