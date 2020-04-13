## Code samples for integrating IoT devices with Helix Sandbox NG

Here you will find code examples to start using the Helix Sandbox with the most used IoT devices.

## About

This tutorial can help you to understand the most popular REST methods used on CEF Context Broker:

#### :one: Creating a Temperature and Humidity Sensor with DHT 22 and NodeMCU ESP8266 v2/v3:

#### About

The code below automatically creates the sensor in the Helix Sandbox and sends the temperature and humidity from DHT 22 to Helix using the restful message with the POST method. Moreover, this code uses the force update to guarantee the storage persistence on the database. 
You can use the Arduino IDE to create the code for your NodeMCU.

```C++
#include <ESP8266WiFi.h> 
#include <ESP8266HTTPClient.h>
#include "DHT.h"
#include <math.h>
#include <Adafruit_Sensor.h>
#define DHTTYPE DHT22   
#define dht_dpin D1
#define LED_BUILTIN 2

const char* orionAddressPath = "IP_HELIX:1026/v2";
const char* deviceID = "ID_DEVICE";

const char* ssid = "SSID"; 
const char* password = "PASSWORD";

//variáveis globais
int medianTemperature;
float totalTemperature;
int medianHumidity;
float totalHumidity;

  
WiFiClient espClient;
HTTPClient http;
DHT dht(dht_dpin, DHTTYPE);

void setup() {
  //setup
  pinMode(LED_BUILTIN, OUTPUT);
  Serial.begin(9600);
  
  //iniciar sensor DHT
  dht.begin();
  
  //iniciar acesso a rede Wi-Fi
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.println("Connecting to WiFi..");
  }

  Serial.println("Connected WiFi network!");
  delay(2000);

  Serial.println("Creating " + String(deviceID) + " entitie...");
  delay(2000);
  //criar entidade no Helix (plug&play) 
  orionCreateEntitie(deviceID);
  
}
 
void loop(){

//zerar variávies
  medianTemperature=0;
  totalTemperature=0;
  medianHumidity=0;
  totalHumidity=0;

//looping para leitura do valor médio de temperatura e umidade
  for(int i = 0; i < 5; i ++){
    totalHumidity  += dht.readHumidity();
    totalTemperature += dht.readTemperature(); 
    Serial.println("COUNT[" + String(i+1) + "] - Total Humidity: " + String(totalHumidity) + " Total Temperature: " + String(totalTemperature));
    delay(5000); //delay 5 seg
    digitalWrite(LED_BUILTIN, HIGH);   // turn the LED on (HIGH is the voltage level)
    delay(250);                       
    digitalWrite(LED_BUILTIN, LOW);    // turn the LED off by making the voltage LOW
    delay(250); 
  };

//cálculo dos valores médios  
  medianHumidity = totalHumidity/5;
  medianTemperature = totalTemperature/5;
  
  Serial.println("Median after 5 reads is Humidity: " + String(medianHumidity) + " Temperature: " + String(medianTemperature));
  
  char msgHumidity[10];
  char msgTemperature[10]; 
  sprintf(msgHumidity,"%d",medianHumidity);
  sprintf(msgTemperature,"%d",medianTemperature);

  //update dos dados no Helix
  Serial.println("Updating data in orion...");
  orionUpdate(deviceID, msgTemperature, msgHumidity);

  //indicação luminosa do envio
  digitalWrite(LED_BUILTIN, HIGH);   // turn the LED on (HIGH is the voltage level)
  delay(500);                       
  digitalWrite(LED_BUILTIN, LOW);    // turn the LED off by making the voltage LOW
  delay(500);                          
  Serial.println("Finished updating data in orion...");
}

//função para a criação da entidade (plug&play)
void orionCreateEntitie(String entitieName) {

    String bodyRequest = "{\"id\": \"" + entitieName + "\", \"type\": \"sensor\", \"temperature\": { \"value\": \"0\", \"type\": \"integer\"},\"humidity\": { \"value\": \"0\", \"type\": \"integer\"}}";
    httpRequest("/entities", bodyRequest);
}

//função de update
void orionUpdate(String entitieID, String temperature, String humidity){

    String bodyRequest = "{\"temperature\": { \"value\": \""+ temperature + "\", \"type\": \"integer\"}, \"humidity\": { \"value\": \""+ humidity +"\", \"type\": \"integer\"}}";
    String pathRequest = "/entities/" + entitieID + "/attrs?options=forcedUpdate";
    httpRequest(pathRequest, bodyRequest);
}

//função request
void httpRequest(String path, String data)
{
  String payload = makeRequest(path, data);

  if (!payload) {
    return;
  }

  Serial.println("##[RESULT]## ==> " + payload);

}

//request
String makeRequest(String path, String bodyRequest)
{
  String fullAddress = "http://" + String(orionAddressPath) + path;
  http.begin(fullAddress);
  Serial.println("Orion URI request: " + fullAddress);

  http.addHeader("Content-Type", "application/json"); 
  http.addHeader("Accept", "application/json"); 
  http.addHeader("fiware-service", "helixiot"); 
  http.addHeader("fiware-servicepath", "/"); 

Serial.println(bodyRequest);
  int httpCode = http.POST(bodyRequest);

  String response =  http.getString();

  Serial.println("HTTP CODE");
  Serial.println(httpCode);
  
  if (httpCode < 0) {
    Serial.println("request error - " + httpCode);
    return "";
  }

  if (httpCode != HTTP_CODE_OK) {
    return "";
  }

  http.end();

  return response;
}
```


