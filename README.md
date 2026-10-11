# IO7 Framework for ESP8266

This is a part of io7 IOT Platform https://github.com/io7lab to help build an IOT device.

With this library, the developer can create an IO7 ESP8266 IOT device which
1. helps configure with the Captive Portal if not configured as in the following picture,
2. or boots with the stored configuration if configured already,
3. connects to the WiFi/IO7 IOT Platform and run the loop function

![Untitled](https://user-images.githubusercontent.com/13171662/221724691-8d1c93be-c4d2-4ad9-8bcf-24bda369efe4.jpg)

# How to use the IO7F8266
You can create a PlatformIO project with the example directory and modify the src/main.cpp for your purpose and build it.

## src/main.cpp 
The following code is the example to use the library. 
```c
#include <Arduino.h>
#include <IO7F8266.h>

String user_html =
    ""
    // USER CODE EXAMPLE : your custom config variable
    // in meta.XXXXX, XXXXX should match to ArduinoJson index to access
    "<p><input type='text' name='meta.yourVar' placeholder='Your Custom Config'>";
;
// for meta.XXXXX, this var is the C variable to hold the XXXXX
int customVar1;
// USER CODE EXAMPLE : your custom config variable

char* ssid_pfix = (char*)"IOTValve";
unsigned long lastPublishMillis = -pubInterval;
const int RELAY = 15;

void publishData() {
    StaticJsonDocument<512> root;
    JsonObject data = root.createNestedObject("d");

    // USER CODE EXAMPLE : command handling
    data["valve"] = digitalRead(RELAY) == 1 ? "on" : "off";
    // USER CODE EXAMPLE : command handling

    serializeJson(root, msgBuffer);
    client.publish(evtTopic, msgBuffer);
}

void handleUserMeta() {
    // USER CODE EXAMPLE : meta data update
    // If any meta data updated on the Internet, it can be stored to local variable to use for the logic
    // in cfg["meta"]["XXXXX"], XXXXX should match to one in the user_html
    customVar1 = cfg["meta"]["yourVar"];
    // USER CODE EXAMPLE
}

void handleUserCommand(char* topic, JsonDocument* root) {
    JsonObject d = (*root)["d"];

    // USER CODE EXAMPLE : status/change update
    // code if any of device status changes to notify the change
    if (d.containsKey("valve")) {
        if (strstr(d["valve"], "on")) {
            digitalWrite(RELAY, HIGH);
        } else {
            digitalWrite(RELAY, LOW);
        }
        lastPublishMillis = -pubInterval;
    }
    // USER CODE EXAMPLE
}

void setup() {
    Serial.begin(115200);
    // USER CODE EXAMPLE : Initialization
    pinMode(RELAY, OUTPUT);
    // USER CODE EXAMPLE

    initDevice();
    // USER CODE EXAMPLE : meta data to local variable
    JsonObject meta = cfg["meta"];
    pubInterval = meta.containsKey("pubInterval") ? meta["pubInterval"] : 0;
    lastPublishMillis = -pubInterval;
    // USER CODE EXAMPLE

    WiFi.mode(WIFI_STA);
    WiFi.begin((const char*)cfg["ssid"], (const char*)cfg["w_pw"]);
    while (WiFi.status() != WL_CONNECTED) {
        delay(500);
        Serial.print(".");
    }
    // main setup
    Serial.printf("\nIP address : ");
    Serial.println(WiFi.localIP());

    // if there is any user Meta Data, implement the funtcion and assign it to userMeta
    userMeta = handleUserMeta;
    // if there is any user Command, implement the funtcion and assign it to userCommand
    userCommand = handleUserCommand;
    set_iot_server();
    iot_connect();
}

void loop() {
    if (!client.connected()) {
        iot_connect();
    }
    // USER CODE EXAMPLE : main loop
    //     you can put any main code here, for example,
    //     the continous data acquistion and local intelligence can be placed here
    // USER CODE EXAMPLE
    client.loop();
    if ((pubInterval != 0) && (millis() - lastPublishMillis > pubInterval)) {
        publishData();
        lastPublishMillis = millis();
    }
}
```

## customization
If additional configuration parameter is needed, you can modify the `user_config_html` for your variable, and handle the variable as shown below. The custom varialbe here is yourVar in the `user_html` and handling in setup code as below

In the global section.
```c
String user_html = ""
    "<p><input type='text' name='meta.pubInterval' placeholder='Publish Interval'>";
```

In the functions.
```c
    JsonObject meta = cfg["meta"];
    pubInterval = meta.containsKey("pubInterval") ? atoi((const char*)meta["pubInterval"]) : 0;
```


## SSL/TLS (full X.509 certificate validation)
Since the version 0.2.0, the library validates the TLS connection to the MQTT broker with a full
X.509 CA certificate (BearSSL) instead of the certificate fingerprint. This works the same way as
the IO7F32 (ESP32) library.

On the config portal, check the **Use SSL** box and save. If SSL is enabled, the portal redirects
to the **Certificate** page (`/check_ssl`) where you can paste the CA certificate in PEM format.
- The certificate is stored as `/ca.pem` in the LittleFS.
- If no certificate is entered (or the field is emptied and saved), the built-in default
  Let's Encrypt intermediate (YE2) is used, so Let's Encrypt-signed brokers work out of the
  box. Brokers signed by any other CA (another LE intermediate, VeriSign, a private CA …)
  need that issuer's certificate pasted here once.
- With SSL enabled the broker is contacted on port 8883, otherwise on plain 1883.
- The TLS record buffers are kept small so the handshake fits the ESP8266 RAM: 4096 bytes if the
  broker supports MFLN (maximum fragment length negotiation), 8192 bytes otherwise. If your
  broker sends a longer certificate chain than these can hold, the handshake fails with
  `ssl connection failed RC=...` on the Serial monitor (it retries every 5 seconds instead of
  crashing); raise the sizes from the sketch with
  `wifiClientSecure.setBufferSizes(16384, 512);` before the first connection attempt on boards
  with more RAM.

Notes and known behaviors (same as IO7F32):
- The SSL check box only writes `ssl=true` when it is checked. Because the portal saves only the
  submitted fields, unchecking it does not clear a previously saved `ssl=true`; use the factory
  reset / reconfiguration flow to turn SSL off.
- Certificate validity (notBefore/notAfter) **is enforced**: before the SSL handshake the device
  syncs its clock from NTP (pool.ntp.org, up to ~10 s) and hands the time to BearSSL. Without
  NTP access the handshake fails with `RC=... Certificate is expired or not yet valid.` and
  retries every 5 seconds.
- Devices configured with an older (0.1.x) fingerprint-based version fall back to plain MQTT on
  port 1883 on the first boot with this version, until they are reconfigured with the SSL
  check box. The fingerprint cannot be converted to a CA certificate, so there is no automatic
  migration.


## dependancy and tips
This library uses LittleFS instead of SPIFFS, and needs PubSubClient, ArduinoJson, ConfigPortal8266.

And if you need to send a long MQTT message, then you can increase the default size of 256 byte by setting the build time variable MQTT_MAX_PACKET_SIZE as below. You may need this, if you want to implement the IR remote, since some appliances would need a long sequence of control signal.

The following is a snnipet of a tested platformio.ini.

```
board_build.filesystem=littlefs 
build_flags = 
	-D MQTT_MAX_PACKET_SIZE=512
lib_deps = 
	knolleary/PubSubClient@^2.8
	bblanchon/ArduinoJson@^6.18.5
	https://github.com/yhur/ConfigPortal8266
	https://github.com/io7lab/IO7F8266
 ```

### More Info.
It handles

1. IO7 IOT Device Management like remote boot, remote factory reset
2. IO7 IOT meta data update
3. Over the Air Firmware Update
4. Configuration reporting
5. And this uses the https://github.com/yhur/ConfigPortal8266 as the WiFi configuration utility.
