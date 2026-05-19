#include <WiFi.h>
#include <HTTPClient.h>

// URL вашего веб-приложения Google Apps Script
const char* googleScriptUrl = "https://google.com";

void logToGoogleSheets(String topic, String direction, String payload, String status, String errorType) {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    
    // Формируем параметры запроса (передаем данные системы)
    String url = String(googleScriptUrl) + 
                 "?topic=" + topic + 
                 "&direction=" + direction + 
                 "&payload=" + payload + 
                 "&status=" + status + 
                 "&error=" + errorType;
                 
    http.begin(url);
    int httpCode = http.GET(); // Отправляем данные
    
    if (httpCode > 0) {
      Serial.println("Данные успешно отправлены в Google Sheets");
    } else {
      Serial.println("Ошибка отправки: " + String(httpCode));
    }
    http.end();
  }
}
#include <PubSubClient.h>

WiFiClient espClient;
PubSubClient mqttClient(espClient);

// Топики для работы
const char* topic_sub1 = "esp32/device/command1";
const char* topic_sub2 = "esp32/device/command2";
const char* topic_pub_telemetry = "esp32/analytics/telemetry";
const char* topic_pub_alerts = "esp32/analytics/alerts";

void setupMQTT() {
  mqttClient.setServer("://hivemq.com", 1883);
  mqttClient.setCallback(mqttCallback); // Функция, которая принимает сообщения
}

// Прием сообщений (Подписка)
void mqttCallback(char* topic, byte* payload, unsigned int length) {
  String message = "";
  for (int i = 0; i < length; i++) {
    message += (char)payload[i];
  }
  // Передаем в UI дисплея и в Google Sheets
  logToGoogleSheets(String(topic), "INCOMING", message, "DELIVERED", "NONE");
}

// Отправка сообщений (Публикация)
void sendTelemetry(String data) {
  if (mqttClient.connected()) {
    mqttClient.publish(topic_pub_telemetry, data.c_str());
    logToGoogleSheets(topic_pub_telemetry, "OUTGOING", data, "SENT", "NONE");
  } else {
    // Если связи нет, фиксируем ошибку переподключения
    logToGoogleSheets(topic_pub_telemetry, "OUTGOING", data, "FAILED", "RECONNECT_TRIGGERED");
  }
}
#include <Adafruit_GFX.h>
#include <Adafruit_ILI9341.h>

#define TFT_CS   15
#define TFT_DC   2
#define TFT_RST  4

Adafruit_ILI9341 tft = Adafruit_ILI9341(TFT_CS, TFT_DC, TFT_RST);

// Переменные для ведения внутренней статистики
int totalReceived = 0;
int totalSent = 0;
int reconnectCount = 0;
String brokerStatus = "DISCONNECTED";

void updateScreen() {
  tft.fillScreen(ILI9341_BLACK); // Очистка экрана
  tft.setCursor(10, 10);
  tft.setTextColor(ILI9341_WHITE);
  tft.setTextSize(2);
  
  // Выводим параметры согласно ТЗ
  tft.print("Broker: "); tft.println(brokerStatus);
  tft.print("Sent msgs: "); tft.println(totalSent);
  tft.print("Recv msgs: "); tft.println(totalReceived);
  tft.print("Reconnects: "); tft.println(reconnectCount);
}
String analyzeNetworkStability(int currentPing, int reconnectsInLastHour) {
  // Простая локальная модель оценки аномалий
  if (reconnectsInLastHour > 5 || currentPing > 500) {
    return "ANOMALY: Unstable connection, high jitter!";
  } else if (currentPing > 200) {
    return "WARNING: Delay is higher than normal.";
  } else {
    return "NORMAL: Connection is stable.";
  }
}
