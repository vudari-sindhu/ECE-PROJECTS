/************************************************************
   PREDICTIVE MAINTENANCE AND HEALTH MONITORING
   FOR INDUSTRIAL MOTOR

   Controller : ESP32
   Sensors    : DHT22 + Vibration + ACS712
   Outputs    : Relay + Active Buzzer
   IoT        : Blynk
 ************************************************************/

#define BLYNK_PRINT Serial

#define BLYNK_TEMPLATE_ID   "YOUR_TEMPLATE_ID"
#define BLYNK_TEMPLATE_NAME "Industrial Motor"
#define BLYNK_AUTH_TOKEN    "YOUR_BLYNK_AUTH_TOKEN"

#include <WiFi.h>
#include <BlynkSimpleEsp32.h>
#include <DHT.h>

/*--------------- WiFi Details ----------------*/

char ssid[] = "YOUR_WIFI_NAME";
char pass[] = "YOUR_WIFI_PASSWORD";

/*--------------- Pin Definitions -------------*/

#define DHT_PIN        4
#define DHT_TYPE       DHT22

#define VIBRATION_PIN  27
#define CURRENT_PIN    34

#define RELAY_PIN      26
#define BUZZER_PIN     25

/*--------------- Thresholds ------------------*/

#define TEMP_LIMIT     70.0
#define CURRENT_LIMIT  20.0

/*--------------- ACS712 ----------------------*/
// ACS712 5A version = 185 mV/A
#define ACS_SENSITIVITY 0.185
#define ADC_REFERENCE   3.3
#define ADC_RESOLUTION  4095.0
#define ACS_ZERO        2.5

/*--------------- Objects ---------------------*/

DHT dht(DHT_PIN, DHT_TYPE);
BlynkTimer timer;

/*--------------- Variables -------------------*/

float temperature = 0.0;
float currentValue = 0.0;

bool vibrationDetected = false;
bool manualRelayState = true;

/*========================================================
                 BLYNK RELAY CONTROL
   V3 controls the motor manually from Blynk
========================================================*/

BLYNK_WRITE(V3)
{
  manualRelayState = param.asInt();

  Serial.print("Manual Relay: ");
  Serial.println(manualRelayState ? "ON" : "OFF");
}

/*========================================================
                    READ SENSORS
========================================================*/

void readSensors()
{
  /*--------------- Temperature ----------------*/

  temperature = dht.readTemperature();

  if (isnan(temperature))
  {
    Serial.println("DHT22 Sensor Error!");
    return;
  }

  /*--------------- Vibration ------------------*/

  // Vibration sensor is ACTIVE LOW
  vibrationDetected = (digitalRead(VIBRATION_PIN) == LOW);

  /*--------------- Current --------------------*/

  int adcValue = analogRead(CURRENT_PIN);

  float voltage = adcValue * (ADC_REFERENCE / ADC_RESOLUTION);

  currentValue = abs((voltage - ACS_ZERO) / ACS_SENSITIVITY);

  /*
     Small readings can occur because of sensor noise.
     Treat very small current values as zero.
  */

  if (currentValue < 0.10)
  {
    currentValue = 0.0;
  }

  /*--------------- Send to Blynk --------------*/

  Blynk.virtualWrite(V0, temperature);
  Blynk.virtualWrite(V1, currentValue);

  if (vibrationDetected)
  {
    Blynk.virtualWrite(V2, 255);
  }
  else
  {
    Blynk.virtualWrite(V2, 0);
  }

  /*--------------- Serial Monitor -------------*/

  Serial.println();
  Serial.println("================================");
  Serial.println("       MOTOR HEALTH STATUS       ");
  Serial.println("================================");

  Serial.print("Temperature : ");
  Serial.print(temperature);
  Serial.println(" °C");

  Serial.print("Current     : ");
  Serial.print(currentValue);
  Serial.println(" A");

  Serial.print("Vibration   : ");

  if (vibrationDetected)
  {
    Serial.println("DETECTED");
  }
  else
  {
    Serial.println("NORMAL");
  }

  /*====================================================
                       FAULT LOGIC
  ====================================================*/

  if (temperature > TEMP_LIMIT ||
      vibrationDetected ||
      currentValue > CURRENT_LIMIT)
  {
    /*--------------- FAULT CONDITION -------------*/

    digitalWrite(RELAY_PIN, HIGH);
    digitalWrite(BUZZER_PIN, HIGH);

    Serial.println();
    Serial.println("!!! FAULT DETECTED !!!");
    Serial.println("MOTOR SHUTDOWN");
  }

  else
  {
    /*--------------- NORMAL CONDITION -------------*/

    if (manualRelayState)
    {
      digitalWrite(RELAY_PIN, HIGH);
    }
    else
    {
      digitalWrite(RELAY_PIN, LOW);
    }

    digitalWrite(BUZZER_PIN, LOW);

    Serial.println();
    Serial.println("✓ MOTOR NORMAL");
  }

  Serial.println("================================");
}

/*========================================================
                         SETUP
========================================================*/

void setup()
{
  Serial.begin(115200);

  delay(1000);

  Serial.println();
  Serial.println("==========================================");
  Serial.println(" PREDICTIVE MOTOR HEALTH MONITORING SYSTEM");
  Serial.println("==========================================");

  /*--------------- Pin Modes ----------------*/

  pinMode(VIBRATION_PIN, INPUT);
  pinMode(CURRENT_PIN, INPUT);

  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  /*--------------- Initial Outputs -----------*/

  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);

  /*--------------- Start DHT -----------------*/

  dht.begin();

  /*--------------- Connect to Blynk -----------*/

  Serial.println("Connecting to WiFi and Blynk...");

  Blynk.begin(BLYNK_AUTH_TOKEN, ssid, pass);

  Serial.println("Blynk Connected!");

  /*--------------- Timer ---------------------*/

  timer.setInterval(2000L, readSensors);
}

/*========================================================
                          LOOP
========================================================*/

void loop()
{
  Blynk.run();
  timer.run();
}
