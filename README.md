#include<WiFi.h>
#include <ESP32Ping.h>
#include <ArduinoJson.h>
StaticJsonDocument<200> doc;
#include <NTPClient.h>
#include <WiFiUdp.h>
WiFiUDP ntpUDP;
NTPClient timeClient(ntpUDP, "0.in.pool.ntp.org");//, 5.5 * 3600);
#include <Wire.h>
#include <RtcDS3231.h>
RtcDS3231<TwoWire> Rtc(Wire);
byte serverPing = false;
unsigned long serverPingTime = 0;
#include "FS.h"
#include "SPIFFS.h"
#include "Arduino.h"

#define FORMAT_SPIFFS_IF_FAILED true

const char* file_1 = "/logfile.txt";
const char* file_2 = "/logfile2.txt";
String file1_data ;
String file2_data ;

#include <HTTPClient.h>
#include <WiFiClient.h>
#include <EEPROM.h>
WiFiServer server(80);
#include <OneWire.h>
#include <DallasTemperature.h>
int R1 = 1000; //TDS
int Ra = 25; //TDS Resistance of powering Pins
/*rouer ssid and pasword */
//String ssid = "F1032"; // Client Mode SSID
//String password = "7835082403"; // Client Mode Password
String ssid = "connected"; // Client Mode SSID
String password = "phone123"; // Client Mode Password
String BLEData;
String CoinData;
/************wifi client read and break strings***************/
String req = "";
String firstData, secondData, thirdData, fourthData, fifthData, sixthData, seventhData, eigthData, ninthData, tenthData, eleventhData ;
String serverData, logData = "";
String machineID, CONTROLLER__IP;
//const char* serverName = "http://192.168.1.7/WATER__ATM__2.0/getData.php";
const char* serverName = "http://www.erpradeep.in/water__controller/get__data.php";
int httpCode;
/*Sensor data */
int tds = 0;
float temp = 0;
float PPMconversion = 0.7;
float TemperatureCoef = 0.019;
float K = 2.88;
/* dispance quantity count */
unsigned long CountNoz1_250ml, CountNoz1_1L, CountNoz1_20L;
unsigned long CountNoz2_250ml, CountNoz2_1L, CountNoz2_20L;
unsigned long CountNoz3_250ml, CountNoz3_1L;
//int CountNoz_3;
//int CountNoz_1;
//int CountNoz_2;
//String CountN1, CountN2, CountN3;
String RF250mlCard1, RF250mlCard2, RF250mlCard3, RF250mlCard4, RF250mlCard5, RF250mlCard6, RF250mlCard7, RF250mlCard8, RF250mlCard9, RF250mlCard10;
String RF1LCard1, RF1LCard2, RF1LCard3, RF1LCard4, RF1LCard5, RF1LCard6, RF1LCard7, RF1LCard8, RF1LCard9, RF1LCard10;
String RF20LCard1, RF20LCard2, RF20LCard3, RF20LCard4, RF20LCard5, RF20LCard6, RF20LCard7, RF20LCard8, RF20LCard9, RF20LCard10;
String DataRF1, DataRF2;

/*eeprom address*/
int address1 = 1;
int address2 = 2;
int address3 = 3;
int address4 = 4;
int address5 = 5;
int address6 = 6;

int Noz1_Cal250ml, Noz1_Cal1L, Noz1_Cal20L;
int Noz2_Cal250ml, Noz2_Cal1L, Noz2_Cal20L;
int Noz3_Cal250ml, Noz3_Cal1L;

/**********Nozzle open interval time************/
unsigned long INTERVAL_nos1 = 100;
unsigned long INTERVAL_nos2 = 100;
unsigned long INTERVAL_nos3 = 100;
unsigned long INTERVAL_TdsTemp = 30000;
unsigned long wifiConnectTime = 10000;
unsigned long wifiScanTime = 10000;
#define ONE_WIRE_BUS 13 //Temp Sensor //GPIO
const int TempProbePossitive = 12; //Temp positive pin //GPIO
/********TDS pins*******/
#define ECPin 36      //GPIO
#define ECPower 32    //GPIO
/********button pins*******/
# define butt1 15   //GPIO
# define butt2 4    //GPIO
# define butt3 5    //GPIO
# define SigLed 35  //GPIO
# define buzz 23    //GPIO
/******relay pins******/
# define relay1 18  //GPIO
# define relay2 19  //GPIO
# define relay3 33  //GPIO
# define powerPin 34 //GPIO
/************Flow sensor pIns**************/
#define FlowSensor1 26   //GPIO
#define FlowSensor2 27   //GPIO
#define FlowSensor3 25   //GPIO
/************RF Reader pins**************/
#define RF1Tx_pin 14   //GPIO
#define RF1Rx_pin 2   //GPIO
#define RF2Tx_pin 16   //GPIO
#define RF2Rx_pin 17   //GPIO

byte powerState = 1;
int powerValue = 0;

float Temperature = 10;
float EC = 0;
float EC25 = 0;
int ppm = 0;
float raw = 0;
float Vin = 5;
float Vdrop = 0;
float Rc = 0;
float buffer = 0;

OneWire oneWire(ONE_WIRE_BUS);// Setup a oneWire instance to communicate with any OneWire devices
DallasTemperature sensors(&oneWire);// Pass our oneWire reference to Dallas Temperature.

String Nosel1 = "off";
String Nosel2 = "off";
String Nosel3 = "off"; // a String to hold incoming data
String inputString = "";
bool stringComplete = false; // whether the string is complete
unsigned long time_1 = 0;
unsigned long time_2 = 0;
unsigned long time_3 = 0;
unsigned long time_4 = 0;
unsigned long currMillis = 0;

int flowCount1 = 0;
int flowCount2 = 0;
int flowCount3 = 0;
int StateFlow1;
int LastStateFlow1;
int StateFlow2;
int LastStateFlow2;
int StateFlow3;
int LastStateFlow3;
unsigned long buzzTime = 0;
byte wifiState = false;
byte buzzState = false;
byte serverBusy = false;
String response = "";
String currentData = "";  // used in core 0 and sendLogData function
void print_time(unsigned long time_millis);

void setup() {
  Serial.begin(9600);
  Serial1.begin(9600, SERIAL_8N1, RF1Tx_pin, RF1Rx_pin); //  Rx Tx
  Serial2.begin(9600, SERIAL_8N1, RF2Tx_pin, RF2Rx_pin); //  Rx Tx
  EEPROM.begin(512);
  delay(2000);
#ifdef ARDUINO_ARCH_ESP32
  Wire.begin(21, 22); //  D21 and D22 on ESP32 sda scl
#else
  Wire.begin();
#endif

  if (!SPIFFS.begin(FORMAT_SPIFFS_IF_FAILED)) {
    Serial.println("SPIFFS Mount Failed");
    return;
  }
  pinMode(TempProbePossitive , OUTPUT );//ditto but for positive
  digitalWrite(TempProbePossitive , HIGH );
  pinMode(ECPin, INPUT);
  pinMode(ECPower, OUTPUT); //Setting pin for sourcing current
  digitalWrite(ECPower , HIGH );
  pinMode(buzz, OUTPUT);
  digitalWrite(buzz, HIGH);
  delay(500);
  digitalWrite(buzz, LOW);
  // ESP.eraseConfig();
  delay(100);// gives sensor time to settle
  sensors.begin();
  delay(100);
  R1 = (R1 + Ra);
  delay(10);
  ReadCountsNoz1();
  delay(10);
  ReadCountsNoz2();
  delay(10);
  ReadCountsNoz3();
  delay(10);
  Read250mlCards();
  delay(10);
  Read1LCards();
  delay(10);
  Read20LCards();
  delay(10);
  Noz1_Cal250ml = EEPROM.read(address1) * 6 ;
  Noz1_Cal1L = EEPROM.read(address2) * 6 ;
  String StringNoz1_Cal20L;
  for (int addr = 10; addr < 16; addr++)
  {
    StringNoz1_Cal20L  += char(EEPROM.read(addr));
  }
  Noz1_Cal20L = StringNoz1_Cal20L.toInt();
  StringNoz1_Cal20L = "";
  Noz2_Cal250ml = EEPROM.read(address3) * 6 ;
  Noz2_Cal1L = EEPROM.read(address4) * 6 ;
  String StringNoz2_Cal20L;
  for (int addr = 16; addr < 22; addr++)
  {
    StringNoz2_Cal20L  += char(EEPROM.read(addr));
  }
  Noz2_Cal20L = StringNoz2_Cal20L.toInt();
  StringNoz2_Cal20L = "";
  Noz3_Cal250ml = EEPROM.read(address5) * 6 ;
  Noz3_Cal1L = EEPROM.read(address6) * 6 ;
  Serial.println("");
  Serial.println("Noz1_Cal250ml: ");
  Serial.println(Noz1_Cal250ml);
  Serial.println("Noz1_Cal1L: ");
  Serial.println(Noz1_Cal1L);
  Serial.println("Noz1_Cal20L: ");
  Serial.println(Noz1_Cal20L);
  Serial.println("Noz2_Cal250ml: ");
  Serial.println(Noz2_Cal250ml);
  Serial.println("Noz2_Cal1L: ");
  Serial.println(Noz2_Cal1L);
  Serial.println("Noz2_Cal20L: ");
  Serial.println(Noz2_Cal20L);
  Serial.println("NozzleCount1_250ml: " + String(CountNoz1_250ml));
  Serial.println("NozzleCount1_1L: " + String(CountNoz1_1L));
  Serial.println("NozzleCount1_20L: " + String(CountNoz1_20L));
  Serial.println("NozzleCount2_250ml: " + String(CountNoz2_250ml));
  Serial.println("NozzleCount2_1L: " + String(CountNoz2_1L));
  Serial.println("NozzleCount2_20L: " + String(CountNoz2_20L));
  pinMode(butt1, INPUT_PULLUP);
  pinMode(butt2, INPUT_PULLUP);
  pinMode(butt3, INPUT_PULLUP);
  pinMode (FlowSensor1, INPUT_PULLUP);
  pinMode (FlowSensor2, INPUT_PULLUP);
  pinMode (FlowSensor3, INPUT_PULLUP);
  digitalWrite(FlowSensor1, HIGH);
  digitalWrite(FlowSensor2, HIGH);
  digitalWrite(FlowSensor3, HIGH);
  pinMode(relay1, OUTPUT);
  pinMode(relay2, OUTPUT);
  pinMode(relay3, OUTPUT);
  pinMode(SigLed, OUTPUT);
  digitalWrite(relay1, LOW);
  digitalWrite(relay2, LOW);
  digitalWrite(relay3, LOW);
  digitalWrite(SigLed, LOW);
  inputString.reserve(200); //inputString size is 200 byte
  WiFi.disconnect();
  delay(10);
  //  WiFi.mode(WiFi_STA);
  WiFi.begin(ssid.c_str(), password.c_str());
  unsigned long current = millis();
  Serial.println("connecting to: " + ssid + "," + password);
  while (WiFi.status() != WL_CONNECTED && millis() < current + 10000) {
    delay(500);
    Serial.print(".");
    wifiState = false;
  }
  delay(100);
  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("wifi connected");
    //    Serial.print(WiFi.localIP());
    //    CONTROLLER__IP = String(WiFi.localIP());
    wifiState = true;
    getMachineID();
    server.begin();
    delay(100);
    pingToServer();
    serverPingTime = millis();
    if (serverPing == true) {
      Serial.println("connected to internet");
    }
    RTC_Valid();
  }
  delay(100);
  GetTempTds();
  powerValue = analogRead(powerPin);
  delay(100);

  delay(500);
  xTaskCreatePinnedToCore(Task2code, "Task2", 10000, NULL, 1, NULL,  0);
  delay(500);
  // attachInterrupt(digitalPinToInterrupt(FlowSensor1), flow1, CHANGE); // Setup Interrupt
  // attachInterrupt(digitalPinToInterrupt(FlowSensor2), flow2, CHANGE); // Setup Interrupt
  // attachInterrupt(digitalPinToInterrupt(FlowSensor3), flow3, CHANGE); // Setup Interrupt

}

void loop() {
  if (millis() > wifiConnectTime) {
    if (WiFi.status() != WL_CONNECTED) {
      WiFi.disconnect();
      WiFi.begin(ssid.c_str(), password.c_str());
      wifiConnectTime = millis() + 2000;
      digitalWrite(SigLed, LOW);
      wifiState = false;
    }
    else {
      wifiState = true;
      digitalWrite(SigLed, HIGH);
    }
  }

  if (serverBusy == false) {
    SerialRead();
    ReadRF1();
    ReadRF2();
  }
  //when serial read string is "Nosel1Start" or butt1 pressed and nosel is off then nosel1 will be started
  if (((inputString == "StartNosel1") || (digitalRead(butt1) == LOW)) && (Nosel1 == "off"))
  {
    // INTERVAL_nos1 = Noz1_Cal250ml;
    Nosel1 = "StartNosel1";
    inputString = "";
    flowCount1 = 0;
    attachInterrupt(digitalPinToInterrupt(FlowSensor1), flow1, CHANGE); // Setup Interrupt
    if (Nosel2 == "off" && Nosel3 == "off") {
      Serial.println("nosel1 started");
    }
  }
  else if (((inputString == "StartNosel2") || (digitalRead(butt2) == LOW)) && (Nosel2 == "off"))
  {
    // INTERVAL_nos2 = Noz2_Cal250ml;
    Nosel2 = "StartNosel2";
    inputString = "";
    flowCount2 = 0;
    attachInterrupt(digitalPinToInterrupt(FlowSensor2), flow2, CHANGE); // Setup Interrupt
    if (Nosel1 == "off" && Nosel3 == "off") {
      Serial.println("Nosel2 started");
    }
  }
  else if (((inputString == "StartNosel3") || (digitalRead(butt3) == LOW)) && (Nosel3 == "off"))
  {
    // INTERVAL_nos3 = Noz3_Cal250ml;
    Nosel3 = "StartNosel3";
    inputString = "";
    flowCount3 = 0;
    attachInterrupt(digitalPinToInterrupt(FlowSensor3), flow3, CHANGE); // Setup Interrupt
    if (Nosel1 == "off" && Nosel2 == "off") {
      Serial.println("nosel3 started");
    }
  }

  if (Nosel1 == "StartNosel1") {
    digitalWrite(relay1, HIGH);
    // flow1();
    if (flowCount1 >= INTERVAL_nos1) {
      digitalWrite(relay1, LOW);
      detachInterrupt(FlowSensor1);
      if (Nosel2 == "off" && Nosel3 == "off") {
        Serial.println("Nosel1 off");
      }
      Nosel1 = "off";
      flowCount1 = 0;
      if (INTERVAL_nos1 == Noz1_Cal250ml ) {
        CountNoz1_250ml++;
        byte CH1 = 1, TYPE = 1;
        RtcDateTime currTime = Rtc.GetDateTime();
        logData += String(currTime) + "/" + String(CH1) + "/" + String(TYPE) + "/" + String(CountNoz1_250ml) + "/" + String(tds) + "/" + String(temp) + ",";
        for (int addr = 0; addr < String(CountNoz1_250ml).length(); addr++)
        {
          EEPROM.write(addr + 420 , String(CountNoz1_250ml)[addr]);
        }
        EEPROM.commit();
      }
      else if (INTERVAL_nos1 == Noz1_Cal1L) {
        CountNoz1_1L++;
        byte CH1 = 1, TYPE = 2;
        RtcDateTime currTime = Rtc.GetDateTime();
        logData += String(currTime) + "/" + String(CH1) + "/" + String(TYPE) + "/" + String(CountNoz1_1L) + "/" + String(tds) + "/" + String(temp) + ",";
        for (int addr = 0; addr < String(CountNoz1_1L).length(); addr++)
        {
          EEPROM.write(addr + 430 , String(CountNoz1_1L)[addr]);
        }
        EEPROM.commit();
      }
      else if (INTERVAL_nos1 == Noz1_Cal20L) {
        CountNoz1_20L++;
        byte CH1 = 1, TYPE = 3;
        RtcDateTime currTime = Rtc.GetDateTime();
        logData += String(currTime) + "/" + String(CH1) + "/" + String(TYPE) + "/" + String(CountNoz1_20L) + "/" + String(tds) + "/" + String(temp) + ",";
        for (int addr = 0; addr < String(CountNoz1_20L).length(); addr++)
        {
          EEPROM.write(addr + 440 , String(CountNoz1_20L)[addr]);
        }
        EEPROM.commit();
      }
    }
  }

  if (Nosel2 == "StartNosel2") {
    digitalWrite(relay2, HIGH);
    if (flowCount2 >= INTERVAL_nos2) {
      digitalWrite(relay2, LOW);
      detachInterrupt(FlowSensor2);
      if (Nosel1 == "off" && Nosel3 == "off") {
        Serial.println("Nosel2 off");
      }
      Nosel2 = "off";
      flowCount2 = 0;
      if (INTERVAL_nos2 == Noz2_Cal250ml ) {
        CountNoz2_250ml++;
        byte CH2 = 2, TYPE = 1;
        //String jsonBuffer = "";
        RtcDateTime currTime = Rtc.GetDateTime();
        //getJson(String(currTime), CH2, TYPE, CountNoz2_250ml, jsonBuffer);
        logData += String(currTime) + "/" + String(CH2) + "/" + String(TYPE) + "/" + String(CountNoz2_250ml) + "/" + String(tds) + "/" + String(temp) + ",";
        for (int addr = 0; addr < String(CountNoz2_250ml).length(); addr++)
        {
          EEPROM.write(addr + 450 , String(CountNoz2_250ml)[addr]);
        }
        EEPROM.commit();
        //sendToServer(jsonBuffer, logData);
        //Serial.println("sent data: " + String(jsonBuffer));
      }
      else if (INTERVAL_nos2 == Noz2_Cal1L) {
        CountNoz2_1L++;
        byte CH2 = 2, TYPE = 2;
        // String jsonBuffer = "";
        RtcDateTime currTime = Rtc.GetDateTime();
        //getJson(String(currTime), CH2, TYPE, CountNoz2_1L, jsonBuffer);
        logData += String(currTime) + "/" + String(CH2) + "/" + String(TYPE) + "/" + String(CountNoz2_1L) + "/" + String(tds) + "/" + String(temp) + ",";
        for (int addr = 0; addr < String(CountNoz2_1L).length(); addr++)
        {
          EEPROM.write(addr + 460 , String(CountNoz2_1L)[addr]);
        }
        EEPROM.commit();
        //sendToServer(jsonBuffer, logData);
        //Serial.println("sent data: " + jsonBuffer);
      }
      else if (INTERVAL_nos2 == Noz2_Cal20L) {
        CountNoz2_20L++;
        byte CH2 = 2, TYPE = 3;
        //String jsonBuffer = "";
        RtcDateTime currTime = Rtc.GetDateTime();
        //getJson(String(currTime), CH2, TYPE, CountNoz2_20L, jsonBuffer);
        logData += String(currTime) + "/" + String(CH2) + "/" + String(TYPE) + "/" + String(CountNoz2_20L) + "/" + String(tds) + "/" + String(temp) + ",";
        for (int addr = 0; addr < String(CountNoz2_20L).length(); addr++)
        {
          EEPROM.write(addr + 470 , String(CountNoz2_20L)[addr]);
        }
        EEPROM.commit();
        //sendToServer(jsonBuffer, logData);
        //Serial.println("sent data: " + jsonBuffer);
      }
    }
  }

  if (Nosel3 == "StartNosel3") {
    digitalWrite(relay3, HIGH);
    //flow3();
    if (flowCount3 >= INTERVAL_nos3) {
      digitalWrite(relay3, LOW);
      detachInterrupt(FlowSensor3);
      if (Nosel1 == "off" && Nosel2 == "off") {
        Serial.println("Nosel3 off");
      }
      Nosel3 = "off";
      flowCount3 = 0;
      if (INTERVAL_nos3 == Noz3_Cal250ml ) {
        CountNoz3_250ml++;
        byte CH3 = 3, TYPE = 1;
        //String jsonBuffer = "";
        RtcDateTime currTime = Rtc.GetDateTime();
        //getJson(String(currTime), CH3, TYPE, CountNoz3_250ml, jsonBuffer);
        logData += String(currTime) + "/" + String(CH3) + "/" + String(TYPE) + "/" + String(CountNoz3_250ml) + "/" + String(tds) + "/" + String(temp) + ",";
        //sendToServer(jsonBuffer, logData);
        //Serial.println("sent data: " + jsonBuffer);
      }
      else if (INTERVAL_nos3 == Noz3_Cal1L) {
        CountNoz3_1L++;
        byte CH3 = 3, TYPE = 2;
        //String jsonBuffer = "";
        RtcDateTime currTime = Rtc.GetDateTime();
        //getJson(String(currTime), CH3, TYPE, CountNoz3_1L, jsonBuffer);
        logData += String(currTime) + "/" + String(CH3) + "/" + String(TYPE) + "/" + String(CountNoz3_1L) + "/" + String(tds) + "/" + String(temp) + ",";
        //sendToServer(jsonBuffer, logData);
        //Serial.println("sent data: " + jsonBuffer);
      }
    }
  }
  if (millis() >= time_4 + INTERVAL_TdsTemp) {
    GetTempTds();
    time_4 = millis();
  }
  //  if (Nosel1 == "start" || Nosel1 == "StartNosel1") {
  //    flow1();
  //  }
  //  if (Nosel2 == "start" || Nosel2 == "StartNosel2") {
  //    flow2();
  //  }
  //  if (Nosel3 == "start" || Nosel3 == "StartNosel3") {
  //    flow3();
  //  }

  //  powerValue = analogRead(powerPin);
  //  if (powerValue < 400 && powerState == 1) {
  //    Serial.println("Power gone");
  //    writeCountsNoz1();
  //    writeCountsNoz2();
  //    writeCountsNoz3();
  //    powerState = 0;
  //  }
  //  else {
  //    if (powerValue > 550) {
  //      powerState = 1;
  //    }
  //  }
  if (millis() > buzzTime + 500 && buzzState == true) {
    digitalWrite(buzz, LOW);
    buzzState = false;
  }
}

/****************End void loop************************//****************End void loop************************//****************End void loop************************//****************End void loop************************/

void Task2code( void * pvParameters ) {
  // Serial.print("Task2 running on core ");
  // Serial.println(xPortGetCoreID());

  for (;;) {
    delay(100);

    if (wifiState == true) {
      WiFiClient client = server.available();
      if (client.connected()) {
        if (client.available() > 0) {
          req = client.readStringUntil('\r');
          Serial.println(req);
          String s = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
          s += "{'TDS':" + String(tds) + ",";
          s += "'TEMP':" + String(temp) + ",";
          s += "'CH1_250ml':" + String(CountNoz1_250ml) + ",";
          s += "'CH1_1L':" + String(CountNoz1_1L) + ",";
          s += "'CH1_20L':" + String(CountNoz1_20L) + ",";
          s += "'CH2_250ml':" + String(CountNoz2_250ml) + ",";
          s += "'CH2_1L':" + String(CountNoz2_1L) + ",";
          s += "'CH2_20L':" + String(CountNoz2_20L) + ",";
          s += "'CH3_250ml':" + String(CountNoz3_250ml) + ",";
          s += "'CH3_1L':" + String(CountNoz3_1L) + "}";
          int addr_start = req.indexOf(' ');
          int addr_end = req.indexOf(' ', addr_start + 1);
          req = req.substring(addr_start + 2, addr_end);
          int ind1 = req.indexOf('/'); //finds location of first ,
          firstData = req.substring(0, ind1); //captures first data of String
          if (firstData == "status") {
            client.print(s);
          }
          else if (firstData == "BLEData") {
            //Serial.println("BLEData");
            // SerialRead();
            String s = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            s += BLEData;
            client.println(s);
          }
          else if (firstData == "stop") {
            String s = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            s += "all nozzle stop";
            client.println(s);
            Serial.println("all nozzle stop by web");
            digitalWrite(relay1, LOW);
            digitalWrite(relay2, LOW);
            digitalWrite(relay3, LOW);
            Nosel1 = "off";
            Nosel2 = "off";
            Nosel3 = "off";
          }
          else if (firstData == "cleareeprom") {
            String s = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            s += "EEPROM clear and all nosel data set to zero";
            client.println(s);
            Serial.println("EEPROM clear and all nosel data set to zero");
            for (int i = 420; i < 480; i++) {
              EEPROM.write(i, 0);
            }
            EEPROM.commit();
            CountNoz1_250ml = CountNoz1_1L = CountNoz1_20L = 0;
            CountNoz2_250ml = CountNoz2_1L = CountNoz2_20L = 0;
          }
          else if (firstData == "ConfigCard250ml") {
            String s = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            s += "ConfigCard250ml: success";
            client.println(s);
            breakData();
            Write250mlCards();   //eeprom write
          }
          else if (firstData == "ConfigCard1L") {
            String s = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            s += "ConfigCard1L: success";
            client.println(s);
            breakData();
            Write1LCards();      //eeprom write
          }
          else if (firstData == "ConfigCard20L") {
            String s = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            s += "ConfigCard20L: success";
            client.println(s);
            breakData();
            Write20LCards();    //eeprom write
          }
          else if (firstData == "HardCal250ml_Noz1") {
            String s = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            s += "HardCal250ml_Noz1: success";
            client.println(s);
            breakData();
            Noz1_Cal250ml = secondData.toInt();
            EEPROM.write(address1, Noz1_Cal250ml / 6);
            EEPROM.commit();
          }
          else if (firstData == "HardCal1L_Noz1") {
            String s = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            s += "HardCal1L_Noz1: success";
            client.println(s);
            breakData();
            Noz1_Cal1L = secondData.toInt();
            EEPROM.write(address2, Noz1_Cal1L / 6);
            EEPROM.commit();
          }
          else if (firstData == "HardCal20L_Noz1") {
            String s = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            s += "HardCal20L_Noz1: success";
            client.println(s);
            breakData();
            Noz1_Cal20L = secondData.toInt();
            for (int i = 0; i < 6; i++) {
              EEPROM.write(10 + i, secondData[i]);
            }
            EEPROM.commit();
          }
          else if (firstData == "HardCal250ml_Noz2") {
            String s = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            s += "HardCal250ml_Noz2: success";
            client.println(s);
            breakData();
            Noz2_Cal250ml = secondData.toInt();
            EEPROM.write(address3, Noz2_Cal250ml / 6);
            EEPROM.commit();
          }
          else if (firstData == "HardCal1L_Noz2") {
            String s = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            s += "HardCal1L_Noz2: success";
            client.println(s);
            breakData();
            Noz2_Cal1L = secondData.toInt();
            EEPROM.write(address4, Noz2_Cal1L / 6);
            EEPROM.commit();
          }
          else if (firstData == "HardCal20L_Noz2") {
            String s = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            s += "HardCal20L_Noz2: success";
            client.println(s);
            breakData();
            Noz2_Cal20L = secondData.toInt();
            for (int i = 0; i < 6; i++) {
              EEPROM.write(16 + i, secondData[i]);
            }
            EEPROM.commit();
          }
          else if (firstData == "HardCal250ml_Noz3") {
            String s = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            s += "HardCal250ml_Noz3: success";
            client.println(s);
            breakData();
            Noz3_Cal250ml = secondData.toInt();
            EEPROM.write(address5, Noz3_Cal250ml / 6);
            EEPROM.commit();
          }
          else if (firstData == "HardCal1L_Noz3") {
            String s = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            s += "HardCal1L_Noz3: success";
            client.println(s);
            breakData();
            Noz3_Cal1L = secondData.toInt();
            EEPROM.write(address6, Noz3_Cal1L / 6);
            EEPROM.commit();
          }
          else if (firstData == "ReadCal") {
            String s = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            s += "{'DurCH1_250ml':" + String(Noz1_Cal250ml) + ",";
            s += "'DurCH1_1L':" + String(Noz1_Cal1L) + ",";
            s += "'DurCH1_20L':" + String(Noz1_Cal20L) + ",";
            s += "'DurCH2_250ml':" + String(Noz2_Cal250ml) + ",";
            s += "'DurCH2_1L':" + String(CountNoz2_1L) + ",";
            s += "'DurCH2_20L':" + String(CountNoz2_20L) + ",";
            s += "'DurCH3_250ml':" + String(Noz3_Cal250ml) + ",";
            s += "'DurCH3_1L':" + String(CountNoz3_1L) + "}";
            client.println(s);
          }
          //******** callibration Nozzle A for 200ml, 1Ltr and 20Ltr**********//
          else if ((firstData == "startCH1-250ml" || firstData == "startCH1-1L" || firstData == "startCH1-20L") && Nosel1 == "off" ) {
            //Serial.println("Nozzle A cal started");
            digitalWrite(relay1, HIGH);
            Nosel1 = "start";
            String response = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            response += "{'STATUS': 'success', 'ACTION': 'started'}";
            client.print(response);
            flowCount1 = 0;
            // flow1();
          }
          else if (firstData == "stopCH1-250ml" && Nosel1 == "start") {
            digitalWrite(relay1, LOW);
            Nosel1 == "off";
            Noz1_Cal250ml = flowCount1;
            EEPROM.write(address1, Noz1_Cal250ml / 6);
            EEPROM.commit();
            String response = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            response += "{'STATUS': 'success', 'ACTION': 'stoped', 'DURATION': " + String(Noz1_Cal250ml) + "}";
            client.print(response);
            flowCount1 = 0;
            Serial.println(Noz1_Cal250ml);
          }
          else if (firstData == "stopCH1-1L" && Nosel1 == "start") {
            // Serial.println("Nozzle A cal stop: ");
            digitalWrite(relay1, LOW);
            Nosel1 == "off";
            Noz1_Cal1L = flowCount1;
            EEPROM.write(address2, Noz1_Cal1L / 6);
            EEPROM.commit();
            String response = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            response += "{'STATUS': 'success', 'ACTION': 'stoped', 'DURATION': " + String(Noz1_Cal1L) + "}";
            client.print(response);
            flowCount1 = 0;
            Serial.println(Noz1_Cal1L);
          }
          else if (firstData == "stopCH1-20L" && Nosel1 == "start") {
            digitalWrite(relay1, LOW);
            Nosel1 == "off";
            Noz1_Cal20L = flowCount1;
            for (int i = 0; i < 6; i++) {
              EEPROM.write(10 + i, String(Noz1_Cal20L)[i]);
            }
            EEPROM.commit();
            String response = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            response += "{'STATUS': 'success', 'ACTION': 'stoped', 'DURATION': " + String(Noz1_Cal20L) + "}";
            client.print(response);
            flowCount1 = 0;
            Serial.println(Noz1_Cal20L);
          }

          //********* callibration Nozzle B for 250ml, 1Ltr and 20Ltr*********//
          else if ((firstData == "startCH2-250ml" || firstData == "startCH2-1L" || firstData == "startCH2-20L") && Nosel2 == "off" ) {
            Serial.println("Nozzle B cal started");
            digitalWrite(relay2, HIGH);
            Nosel2 = "start";
            String response = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            response += "{'STATUS': 'success', 'ACTION': 'started'}";
            client.print(response);
            flowCount2 = 0;
          }
          else if (firstData == "stopCH2-250ml" && Nosel2 == "start") {
            digitalWrite(relay2, LOW);
            Nosel2 = "off";
            Noz2_Cal250ml = flowCount2;
            EEPROM.write(address3, Noz2_Cal250ml / 6);
            EEPROM.commit();
            String response = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            response += "{'STATUS': 'success', 'ACTION': 'stoped', 'DURATION': " + String(Noz2_Cal250ml) + "}";
            client.print(response);
            flowCount2 = 0;
            Serial.println(Noz2_Cal250ml);
          }
          else if (firstData == "stopCH2-1L" && Nosel2 == "start") {
            // Serial.println("Nozzle A cal stop: ");
            digitalWrite(relay2, LOW);
            Nosel2 = "off";
            Noz2_Cal1L = flowCount2;
            EEPROM.write(address4, Noz2_Cal1L / 6);
            EEPROM.commit();
            String response = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            response += "{'STATUS': 'success', 'ACTION': 'stoped', 'DURATION': " + String(Noz2_Cal1L) + "}";
            client.print(response);
            flowCount2 = 0;
            Serial.println(Noz2_Cal1L);
          }
          else if (firstData == "stopCH2-20L" && Nosel2 == "start") {
            digitalWrite(relay2, LOW);
            Nosel2 = "off";
            Noz2_Cal20L = flowCount2;
            for (int i = 0; i < 6; i++) {
              EEPROM.write(16 + i, String(Noz2_Cal20L)[i]);
            }
            EEPROM.commit();
            String response = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            response += "{'STATUS': 'success', 'ACTION': 'stoped', 'DURATION': " + String(Noz2_Cal20L) + "}";
            client.print(response);
            flowCount2 = 0;
            Serial.println(Noz2_Cal20L);
          }

          //********* callibration Nozzle C*********//
          else if ((firstData == "startCH3-250ml" || firstData == "startCH3-1L") && Nosel3 == "off" ) {
            Serial.println("Nozzle B cal started");
            digitalWrite(relay3, HIGH);
            Nosel3 = "start";
            String response = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            response += "{'STATUS': 'success', 'ACTION': 'started'}";
            client.print(response);
            flowCount3 = 0;
          }
          else if (firstData == "stopCH3-250ml" && Nosel3 == "start") {
            digitalWrite(relay3, LOW);
            Nosel3 = "off";
            Noz3_Cal250ml = flowCount3;
            EEPROM.write(address5, Noz3_Cal250ml / 6);
            EEPROM.commit();
            String response = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            response += "{'STATUS': 'success', 'ACTION': 'stoped', 'DURATION': " + String(Noz3_Cal250ml) + "}";
            client.print(response);
            flowCount3 = 0;
            Serial.println(Noz3_Cal250ml);
          }
          else if (firstData == "stopCH3-1L" && Nosel3 == "start") {
            // Serial.println("Nozzle A cal stop: ");
            digitalWrite(relay3, LOW);
            Nosel3 = "off";
            Noz3_Cal1L = flowCount3;
            EEPROM.write(address6, Noz3_Cal1L / 6);
            EEPROM.commit();
            String response = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            response += "{'STATUS': 'success', 'ACTION': 'stoped', 'DURATION': " + String(Noz3_Cal1L) + "}";
            client.print(response);
            flowCount3 = 0;
            Serial.println(Noz3_Cal1L);
          }
          else {
            String response = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n";
            response += "{'STATUS': 'failed', 'ACTION': 'not defined!'}";
            client.print(response);
          }
          // client.stop();
        }
      }
      if (machineID.length() < 5) {
        getMachineID();
      }

      if ((millis() > serverPingTime + 10000) && Nosel1 == "off" && Nosel2 == "off" && Nosel3 == "off") {  // ping server every 2 min
        serverBusy = true;
        pingToServer();
        serverPingTime = millis();
        serverBusy = false;
      }

      if (millis() > currMillis + 1000 && Nosel1 == "off" && Nosel2 == "off" && Nosel3 == "off" && serverPing == true) {
        serverBusy = true;
        currMillis = millis();
        currentData = logData;
        logData = "";
        sendLogFile(SPIFFS);
        serverBusy = false;
      }
    }
    else {
      serverBusy = false;
      if (logData.length() > 4) {
        savedData(logData);
        logData = "";
      }
    }
  }
}


void GetTempTds() {
  //*********Reading Temperature Of Solution *******************//
  //  sensors.requestTemperatures();// Send the command to get temperatures
  //  Temperature = sensors.getTempCByIndex(0); //Stores Value in Variable
  //  if (Temperature > 0) {
  //    temp = Temperature;
  //  }
  //  else
  //    temp = 0;
  //  //Temperature = 20;
  //  //************Estimates Resistance of Liquid ****************//
  //  digitalWrite(ECPower, HIGH);
  //  //delay(1);
  //  raw = analogRead(ECPin);
  //  raw = analogRead(ECPin); // This is not a mistake, First reading will be low beause if charged a capacitor
  //  digitalWrite(ECPower, LOW);
  //  //***************** Converts to EC **************************//
  //  Vdrop = (Vin * raw) / 4096.0;
  //  Rc = (Vdrop * R1) / (Vin - Vdrop);
  //  Rc = Rc - Ra; //acounting for Digital Pin Resitance
  //  EC = 1000 / (Rc * K);
  //  //*************Compensating For Temperaure********************//
  //  EC25  =  EC / (1 + TemperatureCoef * (Temperature - 25.0));
  //  ppm = (EC25) * (PPMconversion * 1000);
  //  if (ppm != tds) {
  //    ppm = (ppm + tds) / 2;
  //  }
  //  if (ppm > 0) {
  //    tds = ppm;
  //  }
  //  else tds = 0;
  tds = 70;
  temp = 25;
}

void wifiScan() {
  String avlWifi;
  if (millis() > wifiScanTime) {
    WiFi.disconnect();
    int n = WiFi.scanNetworks();
    avlWifi = "[";
    for (int i = 0; i < n; ++i) {
      avlWifi += WiFi.SSID(i);
      avlWifi += ",";
    }
    avlWifi += "]";
  }
  Serial.println(avlWifi);
  wifiScanTime = millis() + 10000;
}

void ReadRF1() {
  char income;
  if (Serial1.available() > 1) {
    for (int i = 0; i < 12; i++) {
      income = Serial1.read();
      // if(serverBusy == false){
      DataRF1 += income;
      // }
    }
    Serial.println(DataRF1);
  }
  if ((DataRF1 == RF250mlCard1 || DataRF1 == RF250mlCard2 || DataRF1 == RF250mlCard3 || DataRF1 == RF250mlCard4 || DataRF1 == RF250mlCard5 || DataRF1 == RF250mlCard6 || DataRF1 == RF250mlCard7 || DataRF1 == RF250mlCard8 || DataRF1 == RF250mlCard9 || DataRF1 == RF250mlCard10) && (Nosel1 == "off")) {
    inputString = "StartNosel1";
    INTERVAL_nos1 = Noz1_Cal250ml;
    Serial.println("200ml for nozzle-1");
    digitalWrite(buzz, HIGH);
    buzzState = true;
    buzzTime = millis();
  }
  else if ((DataRF1 == RF1LCard1 || DataRF1 == RF1LCard2 || DataRF1 == RF1LCard3 || DataRF1 == RF1LCard4 || DataRF1 == RF1LCard5 || DataRF1 == RF1LCard6 || DataRF1 == RF1LCard7 || DataRF1 == RF1LCard8 || DataRF1 == RF1LCard9 || DataRF1 == RF1LCard10) && (Nosel1 == "off")) {
    inputString = "StartNosel1";
    INTERVAL_nos1 = Noz1_Cal1L;
    Serial.println("1Ltr for nozzle-1");
    digitalWrite(buzz, HIGH);
    buzzState = true;
    buzzTime = millis();
  }
  else if ((DataRF1 == RF20LCard1 || DataRF1 == RF20LCard2 || DataRF1 == RF20LCard3 || DataRF1 == RF20LCard4 || DataRF1 == RF20LCard5 || DataRF1 == RF20LCard6 || DataRF1 == RF20LCard7 || DataRF1 == RF20LCard8 || DataRF1 == RF20LCard9 || DataRF1 == RF20LCard10) && (Nosel1 == "off")) {
    inputString = "StartNosel1";
    INTERVAL_nos1 = Noz1_Cal20L;
    Serial.println("20Ltr for nozzle-1");
    digitalWrite(buzz, HIGH);
    buzzState = true;
    buzzTime = millis();
  }
  DataRF1 = "";
}
void ReadRF2() {
  char income;
  if (Serial2.available() > 1) {
    for (int i = 0; i < 12; i++) {
      income = Serial2.read();
      //  if(serverBusy == false){
      DataRF2 += income;
      //  }
    }
    Serial.println(DataRF2);
  }
  if ((DataRF2 == RF250mlCard1 || DataRF2 == RF250mlCard2 || DataRF2 == RF250mlCard3 || DataRF2 == RF250mlCard4 || DataRF2 == RF250mlCard5 || DataRF2 == RF250mlCard6 || DataRF2 == RF250mlCard7 || DataRF2 == RF250mlCard8 || DataRF2 == RF250mlCard9 || DataRF2 == RF250mlCard10) && (Nosel2 == "off")) {
    inputString = "StartNosel2";
    INTERVAL_nos2 = Noz2_Cal250ml;
    Serial.println("200ml for nozzle-2");
    digitalWrite(buzz, HIGH);
    buzzState = true;
    buzzTime = millis();
  }
  else if ((DataRF2 == RF1LCard1 || DataRF2 == RF1LCard2 || DataRF2 == RF1LCard3 || DataRF2 == RF1LCard4 || DataRF2 == RF1LCard5 || DataRF2 == RF1LCard6 || DataRF2 == RF1LCard7 || DataRF2 == RF1LCard8 || DataRF2 == RF1LCard9 || DataRF2 == RF1LCard10) && (Nosel2 == "off")) {
    inputString = "StartNosel2";
    INTERVAL_nos2 = Noz2_Cal1L;
    Serial.println("1Ltr for nozzle-2");
    digitalWrite(buzz, HIGH);
    buzzState = true;
    buzzTime = millis();
  }
  else if ((DataRF2 == RF20LCard1 || DataRF2 == RF20LCard2 || DataRF2 == RF20LCard3 || DataRF2 == RF20LCard4 || DataRF2 == RF20LCard5 || DataRF2 == RF20LCard6 || DataRF2 == RF20LCard7 || DataRF2 == RF20LCard8 || DataRF2 == RF20LCard9 || DataRF2 == RF20LCard10) && (Nosel2 == "off")) {
    inputString = "StartNosel2";
    INTERVAL_nos2 = Noz2_Cal20L;
    Serial.println("20Ltr for nozzle-2");
    digitalWrite(buzz, HIGH);
    buzzState = true;
    buzzTime = millis();
  }
  DataRF2 = "";
}

void ReadCountsNoz1() {
  String CH1Count = "" ;
  for (int addr = 420; addr < 430; addr++)
  {
    CH1Count += char(EEPROM.read(addr));
  }
  CountNoz1_250ml = CH1Count.toInt();
  CH1Count = "" ;
  for (int addr = 430; addr < 440; addr++)
  {
    CH1Count += char(EEPROM.read(addr));
  }
  CountNoz1_1L = CH1Count.toInt();
  CH1Count = "" ;
  for (int addr = 440; addr < 450; addr++)
  {
    CH1Count += char(EEPROM.read(addr));
  }
  CountNoz1_20L = CH1Count.toInt();
  CH1Count = "" ;
}

void ReadCountsNoz2() {
  String CH2Count = "" ;
  for (int addr = 450; addr < 460; addr++)
  {
    CH2Count += char(EEPROM.read(addr));
  }
  CountNoz2_250ml = CH2Count.toInt();
  CH2Count = "" ;
  for (int addr = 460; addr < 470; addr++)
  {
    CH2Count += char(EEPROM.read(addr));
  }
  CountNoz2_1L = CH2Count.toInt();
  CH2Count = "" ;
  for (int addr = 470; addr < 480; addr++)
  {
    CH2Count += char(EEPROM.read(addr));
  }
  CountNoz2_20L = CH2Count.toInt();
  CH2Count = "" ;
}

void ReadCountsNoz3() {
  String CH3Count = "" ;
  for (int addr = 480; addr < 490; addr++)
  {
    CH3Count += char(EEPROM.read(addr));
  }
  CountNoz3_250ml = CH3Count.toInt();
  CH3Count = "" ;
  for (int addr = 490; addr < 500; addr++)
  {
    CH3Count += char(EEPROM.read(addr));
  }
  CountNoz3_1L = CH3Count.toInt();
  CH3Count = "" ;
}


void writeCountsNoz1() {
  for (int addr = 0; addr < String(CountNoz1_250ml).length(); addr++)
  {
    EEPROM.write(addr + 420 , String(CountNoz1_250ml)[addr]);
  }
  for (int addr = 0; addr < String(CountNoz1_1L).length(); addr++)
  {
    EEPROM.write(addr + 430 , String(CountNoz1_1L)[addr]);
  }
  for (int addr = 0; addr < String(CountNoz1_20L).length(); addr++)
  {
    EEPROM.write(addr + 440 , String(CountNoz1_20L)[addr]);
  }
  EEPROM.commit();
}
void writeCountsNoz2() {
  for (int addr = 0; addr < String(CountNoz2_250ml).length(); addr++)
  {
    EEPROM.write(addr + 450 , String(CountNoz2_250ml)[addr]);
  }
  for (int addr = 0; addr < String(CountNoz2_1L).length(); addr++)
  {
    EEPROM.write(addr + 460 , String(CountNoz2_1L)[addr]);
  }
  for (int addr = 0; addr < String(CountNoz2_20L).length(); addr++)
  {
    EEPROM.write(addr + 470 , String(CountNoz2_20L)[addr]);
  }
  EEPROM.commit();
}

void writeCountsNoz3() {
  for (int addr = 0; addr < String(CountNoz3_250ml).length(); addr++)
  {
    EEPROM.write(addr + 480 , String(CountNoz3_250ml)[addr]);
  }
  for (int addr = 0; addr < String(CountNoz3_1L).length(); addr++)
  {
    EEPROM.write(addr + 490 , String(CountNoz3_1L)[addr]);
  }
}


void Read250mlCards() {
  for (int addr = 50; addr < 62; addr++)
  {
    RF250mlCard1 += char(EEPROM.read(addr));
  }
  for (int addr = 62; addr < 74; addr++)
  {
    RF250mlCard2 += char(EEPROM.read(addr));
  }
  for (int addr = 74; addr < 86; addr++)
  {
    RF250mlCard3 += char(EEPROM.read(addr));
  }
  for (int addr = 86; addr < 98; addr++)
  {
    RF250mlCard4 += char(EEPROM.read(addr));
  }
  for (int addr = 98; addr < 110; addr++)
  {
    RF250mlCard5 += char(EEPROM.read(addr));
  }
  for (int addr = 110; addr < 122; addr++)
  {
    RF250mlCard6 += char(EEPROM.read(addr));
  }
  for (int addr = 122; addr < 134; addr++)
  {
    RF250mlCard7 += char(EEPROM.read(addr));
  }
  for (int addr = 134; addr < 146; addr++)
  {
    RF250mlCard8 += char(EEPROM.read(addr));
  }
  for (int addr = 146; addr < 158; addr++)
  {
    RF250mlCard9 += char(EEPROM.read(addr));
  }
  for (int addr = 158; addr < 170; addr++)
  {
    RF250mlCard10 += char(EEPROM.read(addr));
  }
  String cards = "{250ml Cards: ";
  cards += RF250mlCard1 + ",";
  cards += RF250mlCard2 + ",";
  cards += RF250mlCard3 + ",";
  cards += RF250mlCard4 + ",";
  cards += RF250mlCard5 + ",";
  cards += RF250mlCard6 + ",";
  cards += RF250mlCard7 + ",";
  cards += RF250mlCard8 + ",";
  cards += RF250mlCard9 + ",";
  cards += RF250mlCard10 + "}";
  Serial.println(cards);
  //return ;
}

void Read1LCards() {
  for (int addr = 182; addr < 194; addr++)
  {
    RF1LCard1 += char(EEPROM.read(addr));
  }
  for (int addr = 194; addr < 206; addr++)
  {
    RF1LCard2 += char(EEPROM.read(addr));
  }
  for (int addr = 206; addr < 218; addr++)
  {
    RF1LCard3 += char(EEPROM.read(addr));
  }
  for (int addr = 218; addr < 230; addr++)
  {
    RF1LCard4 += char(EEPROM.read(addr));
  }
  for (int addr = 230; addr < 242; addr++)
  {
    RF1LCard5 += char(EEPROM.read(addr));
  }
  for (int addr = 242; addr < 254; addr++)
  {
    RF1LCard6 += char(EEPROM.read(addr));
  }
  for (int addr = 254; addr < 266; addr++)
  {
    RF1LCard7 += char(EEPROM.read(addr));
  }
  for (int addr = 266; addr < 278; addr++)
  {
    RF1LCard8 += char(EEPROM.read(addr));
  }
  for (int addr = 278; addr < 290; addr++)
  {
    RF1LCard9 += char(EEPROM.read(addr));
  }
  for (int addr = 290; addr < 302; addr++)
  {
    RF1LCard10 += char(EEPROM.read(addr));
  }
  String cards = "{One_Ltr_Cards: ";
  cards += RF1LCard1 + ",";
  cards += RF1LCard2 + ",";
  cards += RF1LCard3 + ",";
  cards += RF1LCard4 + ",";
  cards += RF1LCard5 + ",";
  cards += RF1LCard6 + ",";
  cards += RF1LCard7 + ",";
  cards += RF1LCard8 + ",";
  cards += RF1LCard9 + ",";
  cards += RF1LCard10 + "}";
  Serial.println(cards);
  //return ;
}

void Read20LCards() {
  for (int addr = 302; addr < 314; addr++)
  {
    RF20LCard1 += char(EEPROM.read(addr));
  }
  for (int addr = 314; addr < 326; addr++)
  {
    RF20LCard2 += char(EEPROM.read(addr));
  }
  for (int addr = 326; addr < 338; addr++)
  {
    RF20LCard3 += char(EEPROM.read(addr));
  }
  for (int addr = 338; addr < 350; addr++)
  {
    RF20LCard4 += char(EEPROM.read(addr));
  }
  for (int addr = 350; addr < 362; addr++)
  {
    RF20LCard5 += char(EEPROM.read(addr));
  }
  for (int addr = 362; addr < 374; addr++)
  {
    RF20LCard6 += char(EEPROM.read(addr));
  }
  for (int addr = 374; addr < 386; addr++)
  {
    RF20LCard7 += char(EEPROM.read(addr));
  }
  for (int addr = 386; addr < 398; addr++)
  {
    RF20LCard8 += char(EEPROM.read(addr));
  }
  for (int addr = 398; addr < 410; addr++)
  {
    RF20LCard9 += char(EEPROM.read(addr));
  }
  for (int addr = 410; addr < 422; addr++)
  {
    RF20LCard10 += char(EEPROM.read(addr));
  }
  String cards = "{20_Ltr_Cards: ";
  cards += RF20LCard1 + ",";
  cards += RF20LCard2 + ",";
  cards += RF20LCard3 + ",";
  cards += RF20LCard4 + ",";
  cards += RF20LCard5 + ",";
  cards += RF20LCard6 + ",";
  cards += RF20LCard7 + ",";
  cards += RF20LCard8 + ",";
  cards += RF20LCard9 + ",";
  cards += RF20LCard10 + "}";
  Serial.println(cards);
  //return ;
}
void breakData() {
  int ind1 = req.indexOf('/');
  int ind2 = req.indexOf('/', ind1 + 1); //finds location of second ,
  int ind3 = req.indexOf('/', ind2 + 1);
  int ind4 = req.indexOf('/', ind3 + 1);
  int ind5 = req.indexOf('/', ind4 + 1);
  int ind6 = req.indexOf('/', ind5 + 1);
  int ind7 = req.indexOf('/', ind6 + 1);
  int ind8 = req.indexOf('/', ind7 + 1);
  int ind9 = req.indexOf('/', ind8 + 1);
  int ind10 = req.indexOf('/', ind9 + 1);
  int ind11 = req.indexOf('/', ind10 + 1);
  secondData = req.substring(ind1 + 1, ind2); //captures second data String
  thirdData = req.substring(ind2 + 1, ind3);
  fourthData = req.substring(ind3 + 1, ind4);
  fifthData = req.substring(ind4 + 1, ind5);
  sixthData = req.substring(ind5 + 1, ind6);
  seventhData = req.substring(ind6 + 1, ind7);
  eigthData = req.substring(ind7 + 1, ind8);
  ninthData = req.substring(ind8 + 1, ind9);
  tenthData = req.substring(ind9 + 1, ind10);
  eleventhData = req.substring(ind10 + 1, ind11);
  //          Serial.println(firstData);
  //          Serial.println(secondData);
  //          Serial.println(thirdData);
  //          Serial.println(fourthData);
  //          Serial.println(fifthData);
  //          Serial.println(sixthData);
  //          Serial.println(seventhData);
  //          Serial.println(eigthData);
  //          Serial.println(ninthData);
  //          Serial.println(tenthData);
  //          Serial.println(eleventhData);
}

void Write250mlCards() {
  for (int addr = 0; addr < secondData.length(); addr++)
  {
    EEPROM.write(addr + 50 , secondData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < thirdData.length(); addr++)
  {
    EEPROM.write(addr + 62 , thirdData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < fourthData.length(); addr++)
  {
    EEPROM.write(addr + 74 , fourthData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < fifthData.length(); addr++)
  {
    EEPROM.write(addr + 86 , fifthData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < sixthData.length(); addr++)
  {
    EEPROM.write(addr + 98 , sixthData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < seventhData.length(); addr++)
  {
    EEPROM.write(addr + 110 , seventhData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < eigthData.length(); addr++)
  {
    EEPROM.write(addr + 122 , eigthData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < ninthData.length(); addr++)
  {
    EEPROM.write(addr + 134 ,  ninthData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < tenthData.length(); addr++)
  {
    EEPROM.write(addr + 146 ,  tenthData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < eleventhData.length(); addr++)
  {
    EEPROM.write(addr + 158,  eleventhData[addr]);
  }
  EEPROM.commit();
}

void Write1LCards() {
  for (int addr = 0; addr < secondData.length(); addr++)
  {
    EEPROM.write(addr + 182 , secondData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < thirdData.length(); addr++)
  {
    EEPROM.write(addr + 194 , thirdData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < fourthData.length(); addr++)
  {
    EEPROM.write(addr + 206 , fourthData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < fifthData.length(); addr++)
  {
    EEPROM.write(addr + 218 , fifthData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < sixthData.length(); addr++)
  {
    EEPROM.write(addr + 230 , sixthData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < seventhData.length(); addr++)
  {
    EEPROM.write(addr + 242 , seventhData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < eigthData.length(); addr++)
  {
    EEPROM.write(addr + 254 , eigthData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < ninthData.length(); addr++)
  {
    EEPROM.write(addr + 266 ,  ninthData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < tenthData.length(); addr++)
  {
    EEPROM.write(addr + 278 ,  tenthData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < eleventhData.length(); addr++)
  {
    EEPROM.write(addr + 290,  eleventhData[addr]);
  }
  EEPROM.commit();
}

void Write20LCards() {
  for (int addr = 0; addr < secondData.length(); addr++)
  {
    EEPROM.write(addr + 302 , secondData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < thirdData.length(); addr++)
  {
    EEPROM.write(addr + 314 , thirdData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < fourthData.length(); addr++)
  {
    EEPROM.write(addr + 326 , fourthData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < fifthData.length(); addr++)
  {
    EEPROM.write(addr + 338 , fifthData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < sixthData.length(); addr++)
  {
    EEPROM.write(addr + 350 , sixthData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < seventhData.length(); addr++)
  {
    EEPROM.write(addr + 362 , seventhData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < eigthData.length(); addr++)
  {
    EEPROM.write(addr + 374 , eigthData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < ninthData.length(); addr++)
  {
    EEPROM.write(addr + 386 ,  ninthData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < tenthData.length(); addr++)
  {
    EEPROM.write(addr + 398 ,  tenthData[addr]);
  }
  EEPROM.commit();
  for (int addr = 0; addr < eleventhData.length(); addr++)
  {
    EEPROM.write(addr + 410,  eleventhData[addr]);
  }
  EEPROM.commit();
}

void SerialRead() {
  if (Serial.available()) {
    inputString = Serial.readStringUntil('\n');
    Serial.println(inputString);
    int ind1 = inputString.indexOf('/'); //finds location of first ,
    String serialFirst = inputString.substring(0, ind1); //captures first data of String
    if (serialFirst == "BLE") {
      int ind2 = inputString.indexOf('/', ind1 + 1);
      BLEData = inputString.substring(ind1 + 1, ind2);
    }
    else if (serialFirst == "Coin") {
      int ind2 = inputString.indexOf('/', ind1 + 1);
      CoinData = inputString.substring(ind1 + 1, ind2);
      if (CoinData == "TwoRupees") {
        inputString = "StartNosel3";
        INTERVAL_nos3 = Noz3_Cal250ml;
        CoinData = "";
      }
      if (CoinData == "fiveRupees") {
        inputString = "StartNosel3";
        INTERVAL_nos3 = Noz3_Cal1L;
        CoinData = "";
      }
    }
  }
}

void flow1() {
  flowCount1++;
  //  Serial.println("flowCount1: " + String(flowCount1));

}
void flow2() {
  flowCount2++;
  // Serial.println("flowCount2: " + String(flowCount2));
}
void flow3() {
  flowCount3++;
  Serial.println("flowCount3: " + String(flowCount3));
}

void sendToServer(String url, String saveData) {
  //if (flag == 1) {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(serverName);
    http.addHeader("Content-Type", "application/json");
    //int httpResponseCode = http.POST("{\"MACHINEID\":\"200033\",\"TDS\":\"23123423\",\"TEMP\":\"24234\",\"TIMESTAMP\":\"4234324\",\"CHANNEL\":\"1\",\"TYPE\":\"250ML\",\"COUNT\":\"9\"}");
    int httpResponseCode = http.POST(url);
    //Serial.println("Data sent");
    String payload = http.getString();
    Serial.print("HTTP Response code: ");
    Serial.println(httpResponseCode);
    DeserializationError error1 = deserializeJson(doc, payload);
    int errorCode = doc["code"];
    String msg = doc["msg"];
    Serial.println("errorCode: " + String(errorCode) + ", " + "msg: " + msg);
    if (httpResponseCode != 200 || errorCode != 200 ) {
      savedData(saveData);
    }
    http.end();
    Serial.println();
  }
  else {
    savedData(saveData);
  }
}


void savedData(String saveData) {
  Serial.println("Data saved to memory");
  appendFile(SPIFFS, file_2, saveData.c_str());
}

void appendFile(fs::FS &fs, const char * path, const char * message) {
  // Serial.printf("Appending to file: %s\n", path);

  File file = fs.open(path, FILE_APPEND);
  if (!file) {
    Serial.println("Failed to open file for appending");
    //    if (BTState == 1) {
    //      BTPrint("Error");
    //    }
    // return;
  }
  if (file.print(message)) {
    Serial.println("Data logged");
    //    if (BTState == 1 && firstData == "saveCard") {
    //      BTPrint("Data was saved successfully");
    //    }
  }
  else {
    Serial.println("Append failed");
  }
  file.close();
}


void readFile(fs::FS &fs, const char * path) {
  //  Serial.printf("Reading file: %s\n", path);

  File file = fs.open(path);
  if (!file) {
    Serial.println("Failed to open file for reading");
    return;
  }
  else {
    for (int i = 0; i < file.size(); i++) //Read upto complete file size
    {
      char c = file.read();
      if (path == file_1) {
        file1_data += c;
      }
      else if (path == file_2) {
        file2_data += c;
      }
    }
    if (file.size() > 10) {
      Serial.println("file Reading done");
    }
    file.close();
  }
}

void deleteFile(fs::FS &fs, const char * path) {
  // Serial.printf("Deleting file: %s\r\n", path);
  if (fs.remove(path)) {
    Serial.println("- file deleted");
    SPIFFS.remove(path);
    //    if (BTState == 1) {
    //      BTPrint("Deletedsendlog");
    //    }
  } else {
    //  Serial.println("- delete failed");
    //    if (BTState == 1) {
    //      BTPrint("Error");
    //    }
  }

}


void RTC_Update() {
  // Do udp NTP lookup, epoch time is unix time - subtract the 30 extra yrs (946684800UL) library expects 2000
  timeClient.update();
  unsigned long epochTime = timeClient.getEpochTime();//-946684800UL;
  Rtc.SetDateTime(epochTime);
}
bool RTC_Valid() {
  bool boolCheck = true;
  if (!Rtc.IsDateTimeValid()) {
    Serial.println("RTC lost confidence in the DateTime!  Updating DateTime");
    boolCheck = false;
    RTC_Update();
  }
  if (!Rtc.GetIsRunning())
  {
    Serial.println("RTC was not actively running, starting now.  Updating Date Time");
    Rtc.SetIsRunning(true);
    boolCheck = false;
    RTC_Update();
  }
}
void getJson(String timeStamp, byte channel, byte Type, unsigned long count, String buffer) {
  StaticJsonDocument<200> dispensing_object;
  dispensing_object["IPADDRESS"] = CONTROLLER__IP;
  dispensing_object["MACHINEID"] = machineID;
  dispensing_object["TDS"] = "70";
  dispensing_object["TEMP"] = "25";
  dispensing_object["TIMESTAMP"] = timeStamp;
  dispensing_object["CHANNEL"] = String(channel);
  dispensing_object["TYPE"] = String(Type);
  dispensing_object["COUNT"] = String(count);
  serializeJson(dispensing_object, buffer); // print to client
}

void getMachineID()
{
  RTC_Valid();
  String BOXID = String(WiFi.macAddress());
  for (int i = 0; i < BOXID.length(); i++) {
    if (BOXID[i] != ':') {
      machineID += BOXID[i];
    }
  }
  CONTROLLER__IP = String() + WiFi.localIP()[0] + "." + WiFi.localIP()[1] + "." + WiFi.localIP()[2] + "." + WiFi.localIP()[3];
  Serial.println("IP Address: " + CONTROLLER__IP);
}


void sendLogFile(fs::FS &fs) {
  //Serial.println("Sending data to server");
  File file = fs.open(file_2);
  if (!file) {
    Serial.println("Failed to open file for reading");
    // return;
  }
  else {
    file2_data = "";
    for (int i = 0; i < file.size(); i++) //Read upto complete file size
    {
      char c = file.read();
      file2_data += c;
      if (file2_data.length() > 3000 && c == ',') {
        StaticJsonDocument<4000> log_object;
        log_object["Action"] = "logData";
        log_object["IPADDRESS"] = CONTROLLER__IP;
        log_object["MACHINEID"] = machineID;
        log_object["logData"] = file2_data;
        String buffer = "";
        serializeJson(log_object, buffer); // print to client
        HTTPClient http;
        http.begin(serverName);
        http.addHeader("Content-Type", "application/json");
        //int httpResponseCode = http.POST("{\"MACHINEID\":\"200033\",\"TDS\":\"23123423\",\"TEMP\":\"24234\",\"TIMESTAMP\":\"4234324\",\"CHANNEL\":\"1\",\"TYPE\":\"250ML\",\"COUNT\":\"9\"}");
        int httpResponseCode = http.POST(buffer);
        String payload = http.getString();
        DeserializationError error1 = deserializeJson(doc, payload);
        int errorCode = doc["code"];
        String msg = doc["msg"];
        if (httpResponseCode == 200 && errorCode == 200) {
          response = "successful";
          file2_data = "";
        }
        http.end();
      }
    }
    file.close();
    if (file2_data.length() > 4) {
      StaticJsonDocument<4000> log_object;
      log_object["Action"] = "logData";
      log_object["IPADDRESS"] = CONTROLLER__IP;
      log_object["MACHINEID"] = machineID;
      log_object["logData"] = file2_data;
      String buffer = "";
      serializeJson(log_object, buffer); // print to client
      HTTPClient http;
      http.begin(serverName);
      http.addHeader("Content-Type", "application/json");
      //int httpResponseCode = http.POST("{\"MACHINEID\":\"200033\",\"TDS\":\"23123423\",\"TEMP\":\"24234\",\"TIMESTAMP\":\"4234324\",\"CHANNEL\":\"1\",\"TYPE\":\"250ML\",\"COUNT\":\"9\"}");
      int httpResponseCode = http.POST(buffer);
      String payload = http.getString();
      DeserializationError error1 = deserializeJson(doc, payload);
      int errorCode = doc["code"];
      String msg = doc["msg"];
      if (httpResponseCode == 200 && errorCode == 200) {
        response = "successful";
        file2_data = "";
      }
      http.end();
      Serial.println("file Reading done");
    }
  }
  if (response == "successful") {
    deleteFile(SPIFFS, file_2);
    response = "";
    Serial.println("DataSent");
  }
  if (currentData.length() > 4) {
    StaticJsonDocument<4000> log_object;
    log_object["Action"] = "logData";
    log_object["IPADDRESS"] = CONTROLLER__IP;
    log_object["MACHINEID"] = machineID;
    log_object["logData"] = currentData;
    String buffer = "";
    serializeJson(log_object, buffer); // print to client
    HTTPClient http;
    http.begin(serverName);
    http.addHeader("Content-Type", "application/json");
    //int httpResponseCode = http.POST("{\"MACHINEID\":\"200033\",\"TDS\":\"23123423\",\"TEMP\":\"24234\",\"TIMESTAMP\":\"4234324\",\"CHANNEL\":\"1\",\"TYPE\":\"250ML\",\"COUNT\":\"9\"}");
    int httpResponseCode = http.POST(buffer);
    String payload = http.getString();
    DeserializationError error1 = deserializeJson(doc, payload);
    int errorCode = doc["code"];
    String msg = doc["msg"];
    if (httpResponseCode == 200 && errorCode == 200) {
      currentData = "";
      Serial.println("DataSent");
    }
    else {
      savedData(currentData);
      currentData = "";
    }
    http.end();
  }
}

void pingToServer() {
  bool success = Ping.ping("www.google.com", 1);
  // bool success = Ping.ping("www.erpradeep.in", 3);
  if (!success) {
    Serial.println("Ping failed");
    serverPing = false;
    if (logData.length() > 4) {
      savedData(logData);
      logData = "";
    }
  }
  else {
    serverPing = true;
    Serial.println("Ping success");
  }
}
