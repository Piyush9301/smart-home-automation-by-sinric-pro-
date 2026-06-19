# smart-home-automation-by-sinric-pro-
// Uncomment the following line to enable serial debug output
//#define ENABLE_DEBUG

#ifdef ENABLE_DEBUG
  #define DEBUG_ESP_PORT Serial
  #define NODEBUG_WEBSOCKETS
  #define NDEBUG
#endif 

#include <Arduino.h>
#include <ESP8266WiFi.h>
#include "SinricPro.h"
#include "SinricProSwitch.h"
#include <map>

#define WIFI_SSID         "Airtel_neer_6562"    
#define WIFI_PASS         "Air@41552"
#define APP_KEY           "7f78d472-10c5-4fc7-8fcc-dbbe7a85f0d4"      // Should look like "de0bxxxx-1x3x-4x3x-ax2x-5dabxxxxxxxx"
#define APP_SECRET        "f1bcee5b-8772-45c6-a9af-8a248a3bbcfa-a73554fa-b430-4a2c-8ce6-2b38d65c49fe"   // Should look like "5f36xxxx-x3x7-4x3x-xexe-e86724a9xxxx-4c4axxxx-3x3x-x5xe-x9x3-333d65xxxxxx"

// Enter the device ID here
#define device_ID_1       "6a32a2f609efd1746c0f0b27"

// Define the GPIO connected with the Relay and switch
#define RelayPin1         5  // D1
#define SwitchPin1        10 // SD3
#define wifiLed           16 // D0

// Comment the following line if you use toggle switches instead of tactile buttons
//#define TACTILE_BUTTON 1

#define BAUD_RATE         9600
#define DEBOUNCE_TIME     250

typedef struct {      
  int relayPIN;
  int flipSwitchPIN;
} deviceConfig_t;

// Main configuration for 1 device
std::map<String, deviceConfig_t> devices = {
    // {deviceId, {relayPIN, flipSwitchPIN}}
    {device_ID_1, { RelayPin1, SwitchPin1 }}
};

typedef struct {      
  String deviceId;
  bool lastFlipSwitchState;
  unsigned long lastFlipSwitchChange;
} flipSwitchConfig_t;

std::map<int, flipSwitchConfig_t> flipSwitches;

void setupRelays() { 
  for (auto &device : devices) {           
    int relayPIN = device.second.relayPIN; 
    pinMode(relayPIN, OUTPUT);             
    digitalWrite(relayPIN, HIGH);
  }
}

void setupFlipSwitches() {
  for (auto &device : devices)  {                     
    flipSwitchConfig_t flipSwitchConfig;              

    flipSwitchConfig.deviceId = device.first;         
    flipSwitchConfig.lastFlipSwitchChange = 0;        
    flipSwitchConfig.lastFlipSwitchState = true;     

    int flipSwitchPIN = device.second.flipSwitchPIN;  

    flipSwitches[flipSwitchPIN] = flipSwitchConfig;   
    pinMode(flipSwitchPIN, INPUT_PULLUP);                   
  }
}

bool onPowerState(String deviceId, bool &state){
  Serial.printf("%s: %s\r\n", deviceId.c_str(), state ? "on" : "off");
  int relayPIN = devices[deviceId].relayPIN; 
  digitalWrite(relayPIN, !state);             
  return true;
}

void handleFlipSwitches() {
  unsigned long actualMillis = millis();                                          
  for (auto &flipSwitch : flipSwitches) {                                         
    unsigned long lastFlipSwitchChange = flipSwitch.second.lastFlipSwitchChange;  

    if (actualMillis - lastFlipSwitchChange > DEBOUNCE_TIME) {                    

      int flipSwitchPIN = flipSwitch.first;                                       
      bool lastFlipSwitchState = flipSwitch.second.lastFlipSwitchState;           
      bool flipSwitchState = digitalRead(flipSwitchPIN);                          
      
      if (flipSwitchState != lastFlipSwitchState) {                               
#ifdef TACTILE_BUTTON
        if (flipSwitchState) {                                                    
#endif      
          flipSwitch.second.lastFlipSwitchChange = actualMillis;                  
          String deviceId = flipSwitch.second.deviceId;                           
          int relayPIN = devices[deviceId].relayPIN;                              
          bool newRelayState = !digitalRead(relayPIN);                            
          digitalWrite(relayPIN, newRelayState);                                  

          SinricProSwitch &mySwitch = SinricPro[deviceId];                        
          mySwitch.sendPowerStateEvent(!newRelayState);                            
#ifdef TACTILE_BUTTON
        }
#endif      
        flipSwitch.second.lastFlipSwitchState = flipSwitchState;                  
      }
    }
  }
}

void setupWiFi(){
  Serial.printf("\r\n[Wifi]: Connecting");
  WiFi.begin(WIFI_SSID, WIFI_PASS);

  while (WiFi.status() != WL_CONNECTED)
  {
    Serial.printf(".");
    delay(250);
  }
  digitalWrite(wifiLed, LOW);
  Serial.printf("connected!\r\n[WiFi]: IP-Address is %s\r\n", WiFi.localIP().toString().c_str());
}

void setupSinricPro(){
  for (auto &device : devices)
  {
    const char *deviceId = device.first.c_str();
    SinricProSwitch &mySwitch = SinricPro[deviceId];
    mySwitch.onPowerState(onPowerState);
  }

  SinricPro.begin(APP_KEY, APP_SECRET);
  SinricPro.restoreDeviceStates(true);
}

void setup(){
  Serial.begin(BAUD_RATE);

  pinMode(wifiLed, OUTPUT);
  digitalWrite(wifiLed, HIGH);

  setupRelays();
  setupFlipSwitches();
  setupWiFi();
  setupSinricPro();
}

void loop(){
  SinricPro.handle();
  handleFlipSwitches();
}
