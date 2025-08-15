# LAB-4-PMW-ED2-2025
// LAB 4 PWM + Servo + Switches + Botones (versión DEBUG)
#include <Arduino.h>
#include <ESP32Servo.h>

// ---------- CONFIG ----------
#define RED_LED_PIN     15
#define GREEN_LED_PIN   2
#define YELLOW_LED_PIN  4

#define SWITCH_COLOR    18  // a GND al activarse
#define SWITCH_BRIGHT   19  // a GND al activarse
#define BUTTON_RIGHT    21  // a GND al activarse
#define BUTTON_LEFT     22  // a GND al activarse

// Si tu LED es de ÁNODO COMÚN pon esto en true
const bool LED_INVERT = false;

// ---------- PWM ----------
#define RED_CH     3
#define GREEN_CH   4
#define YELLOW_CH  5

// ---------- SERVO ----------
#define SERVO_PIN 13
Servo myServo;
int servoPosIndex = 2;
int servoPositions[5] = {0, 45, 90, 135, 180};

// ---------- BRILLO / MODO ----------
const int brightnessLevels[4] = {0, 85, 170, 255};
int currentLevel[3] = {0, 0, 0}; // R, G, Y
int selectedMode = 0;            // 0=R,1=G,2=Y,3=Servo
int previousMode = 0;

// ---------- ANTIRREBOTE ----------
const unsigned long debounceDelay = 50;
bool lastSwitchColor = HIGH, lastSwitchBright = HIGH, lastRightBtn = HIGH, lastLeftBtn = HIGH;
unsigned long tColor=0, tBright=0, tRight=0, tLeft=0, tPrint=0;

// Helpers
inline void writePWM(uint8_t ch, uint8_t v){ ledcWrite(ch, LED_INVERT ? 255 - v : v); }
void refreshLEDs(){
  if (selectedMode == 3) {
    switch (servoPosIndex) {
      case 0: case 4: writePWM(RED_CH,0); writePWM(GREEN_CH,0); writePWM(YELLOW_CH,0); break;
      case 1: writePWM(RED_CH,255); writePWM(GREEN_CH,0); writePWM(YELLOW_CH,0); break;
      case 2: writePWM(RED_CH,0); writePWM(GREEN_CH,255); writePWM(YELLOW_CH,0); break;
      case 3: writePWM(RED_CH,0); writePWM(GREEN_CH,0); writePWM(YELLOW_CH,255); break;
    }
  } else {
    writePWM(RED_CH,   selectedMode==0 ? brightnessLevels[currentLevel[0]] : 0);
    writePWM(GREEN_CH, selectedMode==1 ? brightnessLevels[currentLevel[1]] : 0);
    writePWM(YELLOW_CH,selectedMode==2 ? brightnessLevels[currentLevel[2]] : 0);
  }
}

void setup() {
  Serial.begin(115200);
  delay(200);

  // Entradas con pull-up interno (botones/switches a GND)
  pinMode(SWITCH_COLOR, INPUT_PULLUP);
  pinMode(SWITCH_BRIGHT, INPUT_PULLUP);
  pinMode(BUTTON_RIGHT, INPUT_PULLUP);
  pinMode(BUTTON_LEFT, INPUT_PULLUP);

  // PWM LEDs
  ledcSetup(RED_CH,    5000, 8); ledcAttachPin(RED_LED_PIN, RED_CH);
  ledcSetup(GREEN_CH,  5000, 8); ledcAttachPin(GREEN_LED_PIN, GREEN_CH);
  ledcSetup(YELLOW_CH, 5000, 8); ledcAttachPin(YELLOW_LED_PIN, YELLOW_CH);
  writePWM(RED_CH,0); writePWM(GREEN_CH,0); writePWM(YELLOW_CH,0);

  // Servo
  myServo.attach(SERVO_PIN, 500, 2400); // rango típico
  myServo.write(servoPositions[servoPosIndex]);

  Serial.println("\n=== DEBUG INICIADO ===");
  Serial.println("Pulsa SWITCH_COLOR para cambiar de modo (R,G,Y,Servo).");
  Serial.println("Pulsa SWITCH_BRIGHT para cambiar brillo (en modos R/G/Y).");
  Serial.println("Pulsa RIGHT/LEFT para mover el servo entre 0-45-90-135-180.");
  Serial.println("Si no ves cambios, revisa cableado y que los botones vayan a GND.");
}

void loop() {
  bool switchColor = digitalRead(SWITCH_COLOR);
  bool switchBright = digitalRead(SWITCH_BRIGHT);
  bool rightBtn = digitalRead(BUTTON_RIGHT);
  bool leftBtn = digitalRead(BUTTON_LEFT);

  // Cambio de modo
  if (switchColor == LOW && lastSwitchColor == HIGH && (millis()-tColor)>debounceDelay) {
    previousMode = selectedMode;
    selectedMode = (selectedMode + 1) % 4;
    if (previousMode < 3) currentLevel[previousMode] = 0; // apaga el anterior
    refreshLEDs();
    Serial.print("-> Modo: "); Serial.println(selectedMode==3 ? "SERVO" : (selectedMode==0?"ROJO":selectedMode==1?"VERDE":"AMARILLO"));
    tColor = millis();
  }
  lastSwitchColor = switchColor;

  // Cambio de brillo
  if (selectedMode < 3 && switchBright == LOW && lastSwitchBright == HIGH && (millis()-tBright)>debounceDelay) {
    currentLevel[selectedMode] = (currentLevel[selectedMode] + 1) % 4;
    refreshLEDs();
    Serial.print("Brillo "); Serial.print(selectedMode); Serial.print(" -> nivel "); Serial.print(currentLevel[selectedMode]);
    Serial.print(" (PWM="); Serial.print(brightnessLevels[currentLevel[selectedMode]]); Serial.println(")");
    tBright = millis();
  }
  lastSwitchBright = switchBright;

  // Servo derecha
  if (rightBtn == LOW && lastRightBtn == HIGH && (millis()-tRight)>debounceDelay) {
    if (servoPosIndex < 4) servoPosIndex++;
    myServo.write(servoPositions[servoPosIndex]);
    refreshLEDs();
    Serial.print("Servo -> "); Serial.println(servoPositions[servoPosIndex]);
    tRight = millis();
  }
  lastRightBtn = rightBtn;

  // Servo izquierda
  if (leftBtn == LOW && lastLeftBtn == HIGH && (millis()-tLeft)>debounceDelay) {
    if (servoPosIndex > 0) servoPosIndex--;
    myServo.write(servoPositions[servoPosIndex]);
    refreshLEDs();
    Serial.print("Servo -> "); Serial.println(servoPositions[servoPosIndex]);
    tLeft = millis();
  }
  lastLeftBtn = leftBtn;

  // Log periódico de estados (cada 1 s)
  if (millis() - tPrint > 1000) {
    tPrint = millis();
    Serial.print("Estados: SC="); Serial.print(switchColor);
    Serial.print(" SB="); Serial.print(switchBright);
    Serial.print(" R="); Serial.print(rightBtn);
    Serial.print(" L="); Serial.print(leftBtn);
    Serial.print(" | Modo="); Serial.print(selectedMode);
    Serial.print(" Brillos[R,G,Y]=[");
    Serial.print(brightnessLevels[currentLevel[0]]); Serial.print(",");
    Serial.print(brightnessLevels[currentLevel[1]]); Serial.print(",");
    Serial.print(brightnessLevels[currentLevel[2]]); Serial.print("]");
    Serial.print(" Servo="); Serial.println(servoPositions[servoPosIndex]);
  }

  delay(10);
}
