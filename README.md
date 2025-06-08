/*************************
 *     BIBLIOTHÈQUES     *
 *************************/
#include "RTClib.h"
#include "WiFi.h"
#include "ESPAsyncWebServer.h"
#include <SPI.h>
#include <Arduino.h>
#include <LiquidCrystal_I2C.h>
#include <Preferences.h>
#include <HTTPClient.h>
#include <Wire.h>
#include <Adafruit_INA219.h>
#include <PZEM004Tv30.h>
#include <OneWire.h>
#include <DallasTemperature.h>

/*************************
 *   DÉFINITIONS PINS    *
 *************************/
#define rxPin 4
#define txPin 2
#define ONE_WIRE_BUS 19

#define PZEM_RX_PIN 16          // ESP32 RX (Connecté au TX du PZEM)
#define PZEM_TX_PIN 17          // ESP32 TX (Connecté au RX du PZEM)

#define BAUD_RATE       115200
#define TASK_STACK_SIZE 6000    // 6000 mots = ~24 Ko (suffisant)

/*************************
 *   OBJETS HARDWARE     *
 *************************/
Adafruit_INA219 ina219_1(0x40);      // INA219 #1
OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);

HardwareSerial  Sim800L(1);          // UART1  – SIM800L
PZEM004Tv30     pzem(Serial2, PZEM_RX_PIN, PZEM_TX_PIN); // UART2 – PZEM
AsyncWebServer  server(80);
Preferences     preferences;
RTC_DS1307      rtc;
LiquidCrystal_I2C lcd(0x27, 20, 4);

TaskHandle_t Task1Handle = nullptr;  // Gestion SIM800L (cœur 0)

/*************************
 *   VARIABLES GLOBALES  *
 *************************/
// -- Constantes
String  PHONE = "+2290161103855";
String  GOOGLE_SCRIPT_ID = "AKfycbw3tngankA7tPubA8OyFgaNfCjXJsjuNPcVUC7h-MniTs5F1Ugpf9R6cZuKw4ex4hrg";
const   int minValue = 200;    // Capteur niveau mini
const   int maxValue = 3000;   // Capteur niveau maxi

// -- État du système (partagé entre cœurs) --
volatile float voltage   = 0;
volatile float current   = 0;
volatile float power     = 0;
volatile float energy    = 0;
volatile float frequency = 0;
volatile float pf        = 0;

volatile float temperatureC = 0;
volatile int   waterPercent = 0;

// -- Autres variables (non volatiles) --
String textMessage;
String t1, t2, t3, t4;
unsigned long previousMillisPZEM   = 0;
unsigned long previousMillisPrint  = 0;
unsigned long previousMillisPublish = 0;
int previousDay = -1;
int page = 1;
bool etat = LOW, etat1 = LOW, etat2 = LOW, etat3 = LOW, etat4 = LOW;

/*************************
 *   PROTOTYPES          *
 *************************/
void Task1(void *pvParameters);
void sendSMS(String message);
void sendPostRequest(String message);
void updateSerial();
void GPRS_Connection();
void parseData(String buff);
void initGSM();

/*************************
 *        SETUP          *
 *************************/
void setup() {
    // --- UART0 (USB Série)
    Serial.begin(115200);
    // --- UART1 (SIM800L)
    Sim800L.begin(BAUD_RATE, SERIAL_8N1, rxPin, txPin);
    // --- UART2 (PZEM) déjà configuré par la lib

    // --- I2C
    Wire.begin();

    // --- Périphériques
    if (!ina219_1.begin()) {
        Serial.println("Erreur INA219");
        while (1);
    }
    sensors.begin();
    rtc.begin();
    lcd.init();
    lcd.backlight();

    // --- Préférences
    preferences.begin("variable", false);
    previousDay = preferences.getInt("previousDay", 0);

    // --- GPIO (exemple)
    pinMode(LED_BUILTIN, OUTPUT);

    // --- TÂCHE SIM800L (cœur 0)
    xTaskCreatePinnedToCore(
        Task1,             // Fonction
        "Task1",           // Nom
        TASK_STACK_SIZE,   // Taille pile
        NULL,              // Paramètre
        1,                 // Priorité
        &Task1Handle,      // Handle
        0                  // Cœur
    );

    initGSM();            // Init du module GSM
}

/*************************
 *         LOOP          *
 * (gestion capteurs etc)*
 *************************/
void loop() {

    /*********** 1. Lecture capteurs rapides ***********/
    float busVoltage_1 = ina219_1.getBusVoltage_V();

    int waterLevel = analogRead(25);           // Pin du flotteur
    waterPercent = map(waterLevel, minValue, maxValue, 0, 100);
    waterPercent = constrain(waterPercent, 0, 100);

    sensors.requestTemperatures();
    temperatureC = sensors.getTempCByIndex(0);

    /*********** 2. Lecture PZEM (toutes les 2 s) *******/
    if (millis() - previousMillisPZEM >= 2000) {
        float v = pzem.voltage();
        if (!isnan(v)) {              // Si lecture valide
            voltage   = v;
            current   = pzem.current();
            power     = pzem.power();
            energy    = pzem.energy();
            frequency = pzem.frequency();
            pf        = pzem.pf();
        }
        previousMillisPZEM = millis();
    }

    /*********** 3. Affichage LCD (toutes les 300 ms) ***/
    if (millis() - previousMillisPrint >= 300) {
        DateTime now = rtc.now();
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print(now.timestamp(DateTime::TIMESTAMP_TIME));
        lcd.setCursor(11, 0);
        lcd.print(now.timestamp(DateTime::TIMESTAMP_DATE));
        lcd.setCursor(0, 1);
        lcd.printf("V bat: %.2f V", busVoltage_1);
        lcd.setCursor(0, 2);
        lcd.printf("Fuel : %3d %%", waterPercent);
        lcd.setCursor(0, 3);
        lcd.printf("Temp : %.1f C", temperatureC);
        previousMillisPrint = millis();
    }

    /*********** 4. Gestion boutons / LED etc. **********/
    // ... (votre logique existante sans appel à Sim800L)

    delay(100);   // Relâche un peu le CPU
}

/*************************
 *        TASK1          *
 *   (SIM800L & HTTP)    *
 *************************/
void Task1(void *pvParameters) {
    for (;;) {
        /******** 1. Gestion SMS entrants ********/
        if (Sim800L.available()) {
            textMessage = Sim800L.readString();
            parseData(textMessage);   // Si vous voulez parser +HTTPACTION
            // Exemple : commande par SMS
            if (textMessage.indexOf("Datas1") >= 0) {
                DateTime now = rtc.now();
                String msg = "Date : " + String(now.day()) + "/" + String(now.month()) + "/" + String(now.year()) +
                             "\nHeure : " + String(now.hour()) + ":" + String(now.minute()) + ":" + String(now.second()) +
                             "\nVoltage GE : " + String(voltage) + " V" +
                             "\nCurrent : "   + String(current) + " A" +
                             "\nPower : "     + String(power)   + " W";
                sendSMS(msg);
                textMessage = "";
            }
        }

        /******** 2. Publication HTTP périodique *******/
        if (millis() - previousMillisPublish >= 120000) { // toutes les 120 s
            String msg = "Voltage=" + String(voltage) + "&Current=" + String(current);
            sendPostRequest(msg);      // vers votre PHP
            previousMillisPublish = millis();
        }

        vTaskDelay(pdMS_TO_TICKS(100)); // 100 ms
    }
    vTaskDelete(NULL); // Jamais atteint
}

/*************************
 *    FONCTIONS OUTILS   *
 *************************/
void sendSMS(String message) {
    Sim800L.println("AT+CMGS=\"" + PHONE + "\"");
    delay(100);
    Sim800L.print(message);
    delay(100);
    Sim800L.write(26);   // CTRL-Z
    delay(500);

    sendPostRequest(message);          // Même contenu en HTTP si besoin
}

void sendPostRequest(String message) {
    HTTPClient http;
    String url = "http://fe-store.pro/ulrich.php";

    if (http.begin(url)) {
        http.addHeader("Content-Type", "application/x-www-form-urlencoded");
        String payload = "message=" + message;
        int code = http.POST(payload);
        http.end();
    }
}

/* ---------- Helpers SIM800L ---------- */
void updateSerial() {
    while (Sim800L.available()) {
        Serial.write(Sim800L.read());
    }
    while (Serial.available()) {
        Sim800L.write(Serial.read());
    }
}

void GPRS_Connection() {
    Sim800L.println("AT+CGATT=1");
    delay(100);
    Sim800L.println("AT+SAPBR=3,1,\"CONTYPE\",\"GPRS\"");
    delay(100);
    Sim800L.println("AT+SAPBR=3,1,\"APN\",\"airtelgprs.com\"");
    delay(100);
    Sim800L.println("AT+SAPBR=1,1");
    delay(100);
    Sim800L.println("AT+SAPBR=2,1");
}

void parseData(String buff) {
    buff.trim();
    if (buff.indexOf("+HTTPACTION:") > -1) {
        int idx = buff.indexOf(",");
        String code = buff.substring(idx + 1, idx + 4);
        code.trim();
        if (code == "601") {
            GPRS_Connection();
        }
    }
}

void initGSM() {
    Sim800L.println("AT");
    delay(100);
    Sim800L.println("AT+CMEE=1");
    delay(100);
    Sim800L.println("AT+CMGF=1");
    delay(100);
    Sim800L.println("AT+CNMI=2,2,0,0,0");
    delay(100);
    Sim800L.println("AT+CSQ");
    delay(100);
    Sim800L.println("AT+CBC");
    delay(100);
    GPRS_Connection();
}
