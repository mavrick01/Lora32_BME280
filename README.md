# Lora32_BME280 Configuration

This is a Platform IO configuration using a TTGO Lora32-OLED v1 and a BME280 to send temperature readings over LoraWAN with a final destination being Home Assistant

There are 3 steps to this configuration
1) The Lora32 Configuration (the code above)
2) The TTN Configuration (including the decoder)
3) The Mosquitto Broker Configuration to TTN


## 1 - Lora32 Configuration
This is a small adjustement of the sample BME from the Arduino-Lorawan library. The main change is getting the newer Adafruit BME library to work. 

I decided to use PlatformIO to compile this configuration, but that introduced some interesting problems. *Most importantly* are the changes to **platformio.ini**

```build_flags = 
	-D ARDUINO_LMIC_PROJECT_CONFIG_H_SUPPRESS
	-D CFG_au915=1
	-D CFG_sx1276_radio=1
	-D hal_init=LMICHAL_init
  ```
  
  `ARDUINO_LMIC_PROJECT_CONFIG_H_SUPPRESS` is require when using the MCCI Arduino-Lorawan library for it to ignore the project configs file. I tried to edit the include file, 
  but the arduino-lorawan library does not work, so move the configs here. 
  
*  `CFG_au915` is the regional tag, amend as required. 
*  `CFG_sx1276_radio` is the other definition in the includes file, so adjust according to your chipset (this Lora32's chipset)
*  `hal_init=LMICHAL_init` is required to stop the compiler from having an issue with multiple hal_init files (This one had me for a while)
  
Finally include a `secrets.h` file in your `src` directory. You need to define the following from your TTN configuration.
```
#define DEVEUI { <HEX STRING> } // Remember *lsb*
#define APPEUI { <HEX STRING> } // Remember *lsb*
#define APPKEY { <HEX STRING> } // Default *msb*
```

  ## 2 - TheThingsNetwork Configuration
 
Setting up an application was pretty straight forward. Things to remember are:
* Select LoRaWAN version as `LoRaWAN Specification 1.0.3` 
* For simplicity I used the same number generated for the *DEVEUI* as the *APPEUI* - but do remember you must copy it as *least-significant-bit (lsb)*. I had copied as the default *msb* and then wondered why it did not work
* *APPKEY* is copied as *most-significant-bit (msb)*, so no issues there

Finally, below is the Payload Decoder for the Uplink traffic, to read the elements sent in the *Lora32* script

 ### Payload Decoder
_Formatter type_ -> *Custom Javascript formatter*
 ```
  function decodeUplink(input) {
    var decoded = {};

    decoded.temperature = ((input.bytes[0]<<8)>>>0) + input.bytes[1]
    decoded.temperature = (decoded.temperature / 256.0);

    decoded.humidity = ((input.bytes[2]<<8)>>>0) + input.bytes[3];
    decoded.humidity = (decoded.humidity / 65535.0 ) * 100  
    
    decoded.pressure = ((input.bytes[4]<<16)>>>0) + ((input.bytes[5]<<8)>>>0) + input.bytes[6];
    decoded.pressure = (decoded.pressure / 1.0 );

    return { 
      data: decoded
    }
}
```

## 3 - Hassio Integration

To get the details into Home Assistant, I created a broker connection between **Mosquitto** on my *Hassio RPI* and **TheThingsNetwork**.

This required editing **mosquitto.conf** in the `shared/mosquitto` directory 

```
connection ttn
address <TTN PUBLIC TLS ADDRESS>
bridge_attempt_unsubscribe true
remote_username <TTN USERNAME>
remote_password <TTN API PASSWORD>
try_private false
allow_anonymous true
bridge_capath /etc/ssl/certs
bridge_protocol_version mqttv311

topic v3/# in 0 "" ""
```

The parameters **TTN PUBLIC TLS ADDRESS, TTN USERNAME** and **TTN API PASSWORD** came from the *Integrations->MQTT* in the TheThingsNetwork Application

As I was using TLS, I needed to add the code below. 
```
bridge_capath /etc/ssl/certs
bridge_protocol_version mqttv311
````
You will not need this if you go unencrypted, but as it sends the password in clear texts I would *strongly* recommend you do not and it is so simple to turn on

I am thinking about doing some `topic` transformation, as the *MQTT* message is formated as *v3/\<TTN USERNAME\>/devices/\<APP END DEVICE ID\>*, but have left it for the moment.

### Final Step
The almost forgotten step is to tell *Mosqitto* you want it to read the **mosquitto.conf** file. 

In the **Mosquitto AddOn**, change the *Configuration* to have the following in the *Customize* section:
`active: true`

Remember to **save** in the section where you make the change, and it will prompt you to restart the add on.
