# PROJECT-BMS
#define BLYNK_PRINT Serial
#include <ESP8266WiFi.h>
#include <BlynkSimpleEsp8266.h>

char ssid[] = "Sarvavyapi";
char pass[] = "Sairam#009";
char auth[] = "cLjui-Mq7V4_EzqTGT-YyVV2QKanmajF";

#define RELAY_PIN D5
#define MANUAL_BUTTON_PIN D6
#define MODE_BUTTON_PIN D0

const uint32_t INTERVAL = 5000;
const uint32_t DEBOUNCE_DELAY = 50;

bool relayState = false;
bool autoMode = false;
uint32_t previousMillis = 0;
uint32_t lastDebounceTime = 0;
int lastModeButtonState = HIGH;
int modeButtonState;

#define VPIN_RELAY_CONTROL V1
#define VPIN_MODE_CONTROL V2
#define VPIN_RELAY_STATUS V3
#define VPIN_MODE_STATUS V4

void setup() {
  Serial.begin(115200);
  
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(MANUAL_BUTTON_PIN, INPUT_PULLUP);
  pinMode(MODE_BUTTON_PIN, INPUT_PULLUP);
  
  digitalWrite(RELAY_PIN, HIGH);  // Relay OFF initially
  
  Blynk.begin(auth, ssid, pass);
  
  Serial.println("System initialized in Manual Mode");
  updateBlynkStatus();
}

void loop() {
  Blynk.run();
  checkModeButton();
  
  if (autoMode) {
    handleAutoMode();
  } else {
    handleManualMode();
  }
}

void checkModeButton() {
  int reading = digitalRead(MODE_BUTTON_PIN);
  
  if (reading != lastModeButtonState) {
    lastDebounceTime = millis();
  }
  
  if ((millis() - lastDebounceTime) > DEBOUNCE_DELAY) {
    if (reading != modeButtonState) {
      modeButtonState = reading;
      if (modeButtonState == LOW) {
        toggleMode();
      }
    }
  }
  
  lastModeButtonState = reading;
}

void toggleMode() {
  autoMode = !autoMode;
  
  if (autoMode) {
    previousMillis = millis();
    Serial.println("Switched to Auto Mode");
    
    // When Auto Mode is turned on, relay should automatically toggle and update Blynk
    toggleRelay();  // Automatically update relay when switching modes
  } else {
    Serial.println("Switched to Manual Mode");
  }
  
  // Send auto mode status to Blynk
  Blynk.virtualWrite(VPIN_MODE_CONTROL, autoMode);
  updateBlynkStatus();
}

void handleAutoMode() {
  uint32_t currentMillis = millis();
  if (currentMillis - previousMillis >= INTERVAL) {
    previousMillis = currentMillis;
    toggleRelay();
  }
}

void handleManualMode() {
  static int lastButtonState = HIGH;
  int buttonState = digitalRead(MANUAL_BUTTON_PIN);
  
  if (buttonState != lastButtonState) {
    delay(DEBOUNCE_DELAY);
    if (digitalRead(MANUAL_BUTTON_PIN) == buttonState) {
      if (buttonState == LOW) {
        toggleRelay();
      }
    }
  }
  lastButtonState = buttonState;
}

void toggleRelay() {
  relayState = !relayState;
  digitalWrite(RELAY_PIN, relayState ? LOW : HIGH);
  Serial.println(relayState ? "Relay ON" : "Relay OFF");
  
  // Update Blynk with relay state
  Blynk.virtualWrite(VPIN_RELAY_CONTROL, relayState);
  updateBlynkStatus();
}

void updateBlynkStatus() {
  Blynk.virtualWrite(VPIN_RELAY_STATUS, relayState);
  Blynk.virtualWrite(VPIN_MODE_STATUS, autoMode);
}

BLYNK_WRITE(VPIN_RELAY_CONTROL) {
  if (!autoMode) {
    relayState = param.asInt();
    digitalWrite(RELAY_PIN, relayState ? LOW : HIGH);
    updateBlynkStatus();
  }
}

BLYNK_WRITE(VPIN_MODE_CONTROL) {
  autoMode = param.asInt();
  if (autoMode) {
    previousMillis = millis();
    // Automatically control relay in auto mode
    toggleRelay();
  }
  updateBlynkStatus();
}
