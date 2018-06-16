# smart-door-bell-nodemcu

#include <ArduinoJson.h>
#include <Arduino.h>
#include <ESP8266WiFi.h>
#include <ESP8266WiFiMulti.h>
#include <WebSocketsClient.h>
#include <Hash.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>


ESP8266WiFiMulti WiFiMulti;
WebSocketsClient webSocket;
#define USE_SERIAL Serial1

int clusterId = -1;

char auth[] = "ac7815a4c1564e65a7529def25fb0fee";
char ssid[] = "G6_9986";
char pass[] = "hilihkintil";

LiquidCrystal_I2C lcd(0x3F, 16, 2);

const int BTN_PIN = D3;
const int BZR_PIN = D8;

int reading;
int state = LOW;
int previous = HIGH;
int frequency = 1000;

void buzzerOn()
{
  digitalWrite(BZR_PIN, HIGH);
  tone(BZR_PIN, frequency);
  delay(500);
  noTone(BZR_PIN);
  //delay(500);
  digitalWrite(BZR_PIN, LOW);
  Serial.println("Buzzer On");
}

void buzzerOff()
{
  digitalWrite(BZR_PIN, HIGH);
  
  noTone(BZR_PIN);
  //delay(500);
  digitalWrite(BZR_PIN, LOW);
  Serial.println("Buzzer Off");
}

void pressButton()
{ 
  reading = digitalRead(BTN_PIN);

  // if the input just went from LOW and HIGH and we've waited long enough
  // to ignore any noise on the circuit, toggle the output pin and remember
  // the time
  if (reading == HIGH && previous == LOW) {
    if (state == HIGH)
    {
      state = LOW;
      Serial.println("On");
      buzzerOff();
    }
    else
    {
      state = HIGH; 
      Serial.println("aaaaaaaaaaaa");
      buzzerOn();
      notifyButtonPress();
    }
    Serial.println("zz");
  }
}

void setup() {
  pinMode(BTN_PIN, INPUT);
  lcd.begin(16,2);
  lcd.init();
  
  Serial.begin(9600);
  delay(100);
  Serial.println();
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);
  
  WiFi.begin(ssid, pass);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  // USE_SERIAL.begin(921600);
  USE_SERIAL.begin(115200);

  //Serial.setDebugOutput(true);
  USE_SERIAL.setDebugOutput(true);

  USE_SERIAL.println();
  USE_SERIAL.println();
  USE_SERIAL.println();

  for(uint8_t t = 4; t > 0; t--) {
    USE_SERIAL.printf("[SETUP] BOOT WAIT %d...\n", t);
    USE_SERIAL.flush();
    delay(1000);
  }

  WiFiMulti.addAP(ssid,pass);

  //WiFi.disconnect();
  while(WiFiMulti.run() != WL_CONNECTED) {
    delay(100);
  }

  // server address, port and URL
  webSocket.begin("192.168.43.52", 8181, "/");

  // event handler
  webSocket.onEvent(webSocketEvent);

  // try ever 5000 again if connection has failed
  webSocket.setReconnectInterval(5000);
}


void webSocketEvent(WStype_t type, uint8_t * payload, size_t length) {

  switch(type) {
    case WStype_DISCONNECTED: {
        Serial.printf("[WSc] Disconnected!\n");
      }
      break;
    case WStype_CONNECTED: {
        Serial.printf("[WSc] Connected to url: %s\n", payload);
        registerCluster();
      }
      break;
    case WStype_TEXT: {
        Serial.printf("[WSc] get text: %s\n", payload);
        StaticJsonBuffer<200> jsonBuffer;
        JsonObject& root = jsonBuffer.parseObject((char *)payload);
  
        const char *type = root["type"];
  
        if (strcmp(type, "registration") == 0) {
          clusterId = root["content"]["id"];
        
          Serial.println(type);
          Serial.println(clusterId); 
        }

        if (strcmp(type, "setText") == 0) {
          lcd.clear();
          const char* text = root["content"]["text"];

          Serial.println(text);
          lcd.backlight();
          lcd.clear();
          lcd.setCursor(1, 0);
          lcd.print(text);
          buzzerOn();
        }

        if (strcmp(type, "soundOn") == 0) {
          buzzerOn();
        }
      }
      break;
    case WStype_BIN: {
        Serial.printf("[WSc] get binary length: %u\n", length);
        hexdump(payload, length);
      }
      break;
  }

}

void registerCluster() {
  String payload = "{\"type\": \"register\", \"content\": { \"id\": 69, \"type\": \"cluster\" }}";

  webSocket.sendTXT(payload);
}

void notifyButtonPress() {
  String payload = "{ \"type\": \"buttonPressed\", \"content\": { \"id\": " + String(clusterId) + " } }";

  webSocket.sendTXT(payload);
}

void loop() {
  pressButton();
  webSocket.loop();
  previous = reading;
}