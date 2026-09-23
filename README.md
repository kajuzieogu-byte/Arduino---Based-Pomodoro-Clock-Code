# Arduino---Based-Pomodoro-Clock-Code
This is my personal Project - The Code

WAVESHARE LCD        ARDUINO NANO
---------------------------------
VCC   -------------> 5V
GND   -------------> GND
DIN   -------------> D11 (MOSI)
CLK   -------------> D13 (SCK)
CS    -------------> D10
DC    -------------> D7
RST   -------------> D8
BL    -------------> D9

LED 1 (Study)       ARDUINO NANO
---------------------------------
Anode (+) ---------> D4
Cathode (-) -------> GND through 220Ω resistor

LED 2 (Break)       ARDUINO NANO
---------------------------------
Anode (+) ---------> D5
Cathode (-) -------> GND through 220Ω resistor

BUTTON              ARDUINO NANO
---------------------------------
One side ----------> D6
Other side --------> GND
(using INPUT_PULLUP in code)

POTENTIOMETER (10kΩ)   ARDUINO NANO
---------------------------------
Left pin -------------> GND
Right pin ------------> 5V
Middle (wiper) -------> A0

PIEZO BUZZER         ARDUINO NANO
---------------------------------
Positive (+) -------> D3
Negative (-) -------> GND
________________________________________________________________________________________________________________

//Name: Richmond Ajuzieogu
//Project Name: Pomodoro Clock 
//Date: 04/06/2026

#include <SPI.h>
#include <Adafruit_GFX.h>
#include <Adafruit_ST7789.h>

// Pin definitions (LCD)
#define TFT_CS     10
#define TFT_DC     7
#define TFT_RST    8
#define TFT_BL     9

// Hardware pins
#define LED_STUDY  4
#define LED_BREAK  5
#define BUTTON     6
#define POT_PIN    A0
#define BUZZER_PIN 3

Adafruit_ST7789 tft = Adafruit_ST7789(TFT_CS, TFT_DC, TFT_RST);

// Timer variables
int seconds = 0;
int minutes = 0;
int count = 0;

// Settings
int study_minutes = 25;
const int short_break_minutes = 5;
const int long_break_minutes = 15;
const int repeats = 4;

int break_duration;

// Colors
#define BG_COLOR     ST77XX_BLACK
#define STUDY_COLOR  ST77XX_GREEN
#define BREAK_COLOR  ST77XX_BLUE
#define TEXT_COLOR   ST77XX_WHITE
#define BAR_BG       ST77XX_BLACK

// ---------- ACTIVE BUZZER FUNCTIONS ----------
void beepOnce(int duration) {
  digitalWrite(BUZZER_PIN, HIGH);
  delay(duration);
  digitalWrite(BUZZER_PIN, LOW);
  delay(80);
}

void beepStartStudy() {
  beepOnce(120);
  delay(80);
  beepOnce(120);
}

void beepStartBreak() {
  beepOnce(250);
}

void beepDone() {
  beepOnce(150);
  delay(100);
  beepOnce(150);
  delay(100);
  beepOnce(350);
}

// ---------------------- Slider ----------------------
void drawSlider(int minutes) {
  int x = 20;
  int y = 220;
  int width = 200;
  int height = 10;

  int fillWidth = map(minutes, 5, 60, 0, width);

  tft.fillRect(x, y, width, height, BAR_BG);
  tft.fillRect(x, y, fillWidth, height, ST77XX_GREEN);
  tft.drawRect(x, y, width, height, ST77XX_WHITE);
}

void drawProgressBar(int progress, int total, uint16_t color) {
  int width = map(progress, 0, total, 0, 200);
  tft.fillRect(20, 200, 200, 10, BAR_BG);
  tft.fillRect(20, 200, width, 10, color);
}

void drawTimer(int m, int s, const char* label, uint16_t color) {
  tft.fillScreen(BG_COLOR);

  tft.setTextSize(2);
  tft.setTextColor(color);
  tft.setCursor(40, 40);
  tft.print(label);

  tft.setTextSize(4);
  tft.setTextColor(TEXT_COLOR);
  tft.setCursor(30, 100);

  if (m < 10) tft.print("0");
  tft.print(m);
  tft.print(":");
  if (s < 10) tft.print("0");
  tft.print(s);
}

void updateStudyTime() {
  int potValue = analogRead(POT_PIN);

  if (potValue < 50) {
    study_minutes = 25;
    return;
  }

  int rawMinutes = map(potValue, 0, 1023, 5, 60);
  study_minutes = (rawMinutes / 5) * 5;
}

void waitForButton() {
  while (digitalRead(BUTTON) == HIGH) {
  }
  delay(200);
}

void runTimer(int totalMinutes, const char* label, uint16_t color, bool isStudy) {

  digitalWrite(LED_STUDY, isStudy ? HIGH : LOW);
  digitalWrite(LED_BREAK, isStudy ? LOW : HIGH);

  if (isStudy) {
    beepStartStudy();
  } else {
    beepStartBreak();
  }

  minutes = 0;

  while (minutes < totalMinutes) {
    seconds = 0;

    while (seconds < 60) {

      drawTimer(minutes, seconds, label, color);

      int totalSeconds = totalMinutes * 60;
      int currentSeconds = minutes * 60 + seconds;

      drawProgressBar(currentSeconds, totalSeconds, color);

      delay(1000);
      seconds++;
    }
    minutes++;
  }
}

void setup() {
  pinMode(BUTTON, INPUT_PULLUP);
  pinMode(LED_STUDY, OUTPUT);
  pinMode(LED_BREAK, OUTPUT);
  pinMode(POT_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  digitalWrite(LED_STUDY, LOW);
  digitalWrite(LED_BREAK, LOW);
  digitalWrite(BUZZER_PIN, LOW);

  tft.init(240, 320);
  tft.setRotation(1);

  pinMode(TFT_BL, OUTPUT);
  digitalWrite(TFT_BL, HIGH);

  tft.fillScreen(BG_COLOR);

  while (digitalRead(BUTTON) == HIGH) {

    updateStudyTime();

    tft.fillRect(0, 80, 240, 200, BG_COLOR);

    tft.setTextSize(2);
    tft.setTextColor(TEXT_COLOR);
    tft.setCursor(30, 80);
    tft.print("Set Study Time");

    tft.setTextSize(3);
    tft.setTextColor(ST77XX_YELLOW);
    tft.setCursor(60, 130);
    tft.print(study_minutes);
    tft.print(" min");

    drawSlider(study_minutes);

    delay(150);
  }

  delay(200);
  beepOnce(120);
}

void loop() {
  count = 0;

  updateStudyTime();

  while (count < repeats) {

    runTimer(study_minutes, "Study Time", STUDY_COLOR, true);

    if (count == repeats - 1) {
      break_duration = long_break_minutes;
      runTimer(break_duration, "Long Break", BREAK_COLOR, false);
    } else {
      break_duration = short_break_minutes;
      runTimer(break_duration, "Short Break", BREAK_COLOR, false);
    }

    count++;
  }

  digitalWrite(LED_STUDY, LOW);
  digitalWrite(LED_BREAK, LOW);

  tft.fillScreen(BG_COLOR);
  tft.setCursor(50, 120);
  tft.setTextSize(3);
  tft.setTextColor(ST77XX_YELLOW);
  tft.print("DONE!");

  beepDone();
  delay(5000);
}// ---------------------- SETUP ----------------------
void setup() {
  pinMode(BUTTON, INPUT_PULLUP);
  pinMode(LED_STUDY, OUTPUT);
  pinMode(LED_BREAK, OUTPUT);
  pinMode(POT_PIN, INPUT);

  tft.init(240, 320);
  tft.setRotation(1);

  pinMode(TFT_BL, OUTPUT);
  digitalWrite(TFT_BL, HIGH);

  tft.fillScreen(BG_COLOR);

  // -------- NEW: INTERACTIVE TIME SELECTION --------
  while (digitalRead(BUTTON) == HIGH) {

    updateStudyTime();

    tft.fillRect(0, 80, 240, 200, BG_COLOR);

    tft.setTextSize(2);
    tft.setTextColor(TEXT_COLOR);
    tft.setCursor(30, 80);
    tft.print("Set Study Time");

    tft.setTextSize(3);
    tft.setTextColor(ST77XX_YELLOW);
    tft.setCursor(60, 130);
    tft.print(study_minutes);
    tft.print(" min");

    drawSlider(study_minutes);

    delay(150);
  }

  delay(200);
}

// ---------------------- LOOP ----------------------
void loop() {
  count = 0;

  // Ensure correct value before starting
  updateStudyTime();

  while (count < repeats) {

    runTimer(study_minutes, "Study Time", STUDY_COLOR, true);

    if (count == repeats - 1) {
      break_duration = long_break_minutes;
      runTimer(break_duration, "Long Break", BREAK_COLOR, false);
    } else {
      break_duration = short_break_minutes;
      runTimer(break_duration, "Short Break", BREAK_COLOR, false);
    }

    count++;
  }

  tft.fillScreen(BG_COLOR);
  tft.setCursor(50, 120);
  tft.setTextSize(3);
  tft.setTextColor(ST77XX_YELLOW);
  tft.print("DONE!");

  delay(5000);
}
