# Calidad-del-aire-
Prototipo de calidad del aire para medir las partículas que pueden poner en riesgo su salud. 
Se utiliaran los siguientes sensores para su funcionaiento, conectados a una ESP32 
MQ 135, MQ7, CJMCU-811 
![image](https://github.com/user-attachments/assets/89e6ce05-8258-4da3-b4e7-ec03abbb8ffe) "MQ7"
![image](https://github.com/user-attachments/assets/28997ac6-a93f-4064-b0d1-ca097fb56716) "MQ135"
![image](https://github.com/user-attachments/assets/b7cf2345-07da-4960-a1b6-c8c39e14f9f4) "CJMCU-811"
![Imagen de WhatsApp 2024-10-23 a las 10 43 20_0d7f8ef0](https://github.com/user-attachments/assets/981c3d31-985d-458b-b07b-2706eef76fb0) "Diagrama de conexiones de los sensores con la ESP32"
CODIGO DE CONEXION (ARDUINO) (desargar la libreria de Adafruit_CCS811.h)
#define SECRET_SSID "####"    // replace MySSID with your WiFi network name
#define SECRET_PASS "####" // replace MyPassword with your WiFi password

#define SECRET_CH_ID 2672677      // replace 0000000 with your channel number
#define SECRET_WRITE_APIKEY "######"   // replace XYZ with your channel write API Key
//#define TS_ENABLE_SSL // For HTTPS SSL connection

//#include <WiFiClientSecure.h>
#include <WiFi.h>
#include <WiFiClient.h> //remplazo
#include "secrets.h"
#include "ThingSpeak.h" // always include thingspeak header file after other header files and custom macros
//mjc....
#include <Wire.h>
#include "Adafruit_CCS811.h"
Adafruit_CCS811 ccs;



char ssid[] = SECRET_SSID;   // your network SSID (name) 
char pass[] = SECRET_PASS;   // your network password
int keyIndex = 0;            // your network key Index number (needed only for WEP)
//WiFiClientSecure  client;
WiFiClient client; //remplazo
unsigned long myChannelNumber = SECRET_CH_ID;
const char * myWriteAPIKey = SECRET_WRITE_APIKEY;

//humo
const int analogPin = 34;

// Initialize our values
const int MQ7_PIN = 32; 
String myStatus = "";

//const char * fingerprint = SECRET_SHA1_FINGERPRINT;

void setup() {
  Serial.begin(115200);  //Initialize serial
  while (!Serial) {
    ; // wait for serial port to connect. Needed for Leonardo native USB port only
  }  
  WiFi.mode(WIFI_STA); 
  ThingSpeak.begin(client);  // Initialize ThingSpeak
  
  // MQ7 Configurar el pin como entrada 
  pinMode(MQ7_PIN, INPUT);
  //HUMO
  pinMode(analogPin, INPUT);

  //cmc....
  // Iniciar la comunicación I2C
  Wire.begin(21, 22); // Configura los pines SDA (21) y SCL (22)

  // Iniciar el sensor
  if (!ccs.begin()) {
    Serial.println("No se pudo encontrar un sensor CCS811. Verifica la conexión!");
    while (1);
  }

  // Esperar a que el sensor esté listo
  while (!ccs.available());


}



void loop() {

 // Connect or reconnect to WiFi
  if(WiFi.status() != WL_CONNECTED){
    Serial.print("Attempting to connect to SSID: ");
    Serial.println(SECRET_SSID);
    while(WiFi.status() != WL_CONNECTED){
      WiFi.begin(ssid, pass);  // Connect to WPA/WPA2 network. Change this line if using open or WEP network
      Serial.print(".");
      delay(5000);     
    } 
    Serial.println("\nConnected.");
  }

  int sensorValue = analogRead(MQ7_PIN);
  float voltage = sensorValue * (3.3 / 4095.0); // Convertir a voltaje considerando resolución ADC y Vcc del ESP32
    // Mostrar el valor en la consola serie
  Serial.print("Valor del sensor: ");
  Serial.print(sensorValue);
  Serial.print("\tVoltaje: ");
  Serial.print(voltage);
  Serial.println(" V");
  
  //humo
  int analogValue = analogRead(analogPin);
  Serial.print("Valor humo");
  Serial.print(analogValue);
  delay(1000);

  //cmc....
  if (ccs.available()) {
    if (!ccs.readData()) {
      Serial.print("CO2: ");
      Serial.print(ccs.geteCO2());
      Serial.print(" ppm, TVOC: ");
      Serial.print(ccs.getTVOC());
      Serial.println(" ppb");
    } else {
      Serial.println("Error al leer los datos del sensor!");
    }
  }
//fin cmc...

  // set the fields with the values
  //mq7
  ThingSpeak.setField(1, sensorValue);
  ThingSpeak.setField(2, voltage);
    //humo
  ThingSpeak.setField(3, analogValue);
  //cmc...
  ThingSpeak.setField(4, ccs.geteCO2());
  ThingSpeak.setField(4, ccs.getTVOC());


  
    // set the status
  ThingSpeak.setStatus(myStatus);
  
  // write to the ThingSpeak channel
  int x = ThingSpeak.writeFields(myChannelNumber, myWriteAPIKey);
  if(x == 200){
    Serial.println("Channel update successful.");
  }
  else{
    Serial.println("Problem updating channel. HTTP error code " + String(x));
  }
 
    
  delay(5000); // Wait 20 seconds to update the channel again
}

