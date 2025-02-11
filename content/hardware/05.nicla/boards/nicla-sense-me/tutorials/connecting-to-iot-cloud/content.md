---
title: Connecting the Nicla Sense ME to IoT Cloud
tags: [IoT, Cloud, IDE, Create ]
description: This tutorial shows you how to set up the Arduino Nicla Sense ME and the Arduino Portenta H7 to connect to Arduino Cloud IoT and upload sensor data.
difficulty: easy
author: Ernesto Lopez, Pablo Marquínez
libraries:
  - name: Arduino BHY2
    url: https://www.github.com/Arduino-libraries/Arduino_BHY2
  - name: Arduino BHY2Host
    url: https://www.github.com/Arduino-libraries/Arduino_BHY2Host
hardware:
  - hardware/05.nicla/boards/nicla-sense-me
software:
  - iot-cloud
  - web-editor
  - ide-v1
  - ide-v2
---

## Overview

In this tutorial you will learn how to upload data from the Nicla module to the IoT Cloud. You will use the **Portenta H7** to interface with the **Nicla Sense ME** using the eslov connector and upload the data using the Portenta Wireless capabilities.

## Goals

- How to configure the Portenta H7 to read temperature values from the Nicla Sense using the ESLOV connector.
- How to Connect the Portenta H7 to the IoT cloud
- Publish the temperature values that we obtained from the Nicla board to the Arduino IoT Cloud.

### Required Hardware and Software

- Portenta H7 board
- Nicla sense ME board
- Eslov cable (included with the Nicla Sense ME)
- USB-C to USB-A / C depending on your hardware (Portenta H7)
- USB-A to micro USB-A (Nicla Sense ME)
- Wi-fi Access point with access to the internet
- Arduino Create account

## Instructions

### Hardware Connection

For the hardware setup just connect the Nicla board to the Portenta H7 using the Eslov cable like in the illustration below. Then connect the Portenta H7 to your computer using an USB-C cable.

![Nicla connection](assets/eslov-connection.svg)

### Setup Eslov Communication with the NICLA Board

There are three ways to read from the on-board sensors:

1. Read the sensors directly from Nicla Sense ME in standalone mode.
2. Read sensor values through Bluetooth® Low Energy
3. Read sensor values through UART by connecting an ESLOV cable.

For further tips on how to operate the Nicla module check the [cheat sheet.](https://docs.arduino.cc/tutorials/nicla-sense-me/cheat-sheet#sensor-data-over-eslov)

### Client Board (Nicla Sense ME)

The **Nicla Sense ME** will be listening to the Host board to send back the required data, this is all automated by the libraries **Arduino_BHY2** and **Arduino_BHY2Host**

The code its available inside the examples provided with the **Arduino_BHY2** Library. Open it by going to **Examples -> Arduino_BHY2 -> App.ino**

This is the code, which initialize the sensors, and maintain the communication

```cpp
/* 
 * Use this sketch if you want to control nicla from 
 * an external device acting as a host.
 * Here, nicla just reacts to external stimuli coming from
 * the eslov port or through Bluetooth® Low Energy 
*/

#include "Arduino.h"
#include "Arduino_BHY2.h"

void setup(){
    BHY2.begin(NICLA_I2C, NICLA_VIA_ESLOV);
}

void loop(){
    // Update and then sleep X ms
    BHY2.update(100);
}
```

### Set up the IoT Cloud

To configure the IoT cloud you can follow this [tutorial.](https://docs.arduino.cc/cloud/iot-cloud/tutorials/iot-cloud-getting-started)

Create a new thing at <https://create.arduino.cc/iot/things>, we called it "PRO - Portenta and Nicla". You will need to attach the Portenta H7 as a new device to your **Thing setup.** After that go to **Variables** and click the **add** button and select a float variable called **temperature** to store the temperature readings.

![Arduino IoT Cloud - Thing Setup](assets/IoTCloud-thingSetup.png)

***Don't forget to add your Wi-Fi SSID name, and its password (you can do that inside the Thing setup tab) to be able to connect to the Arduino IoT Cloud***

### Host Board: Edit the Cloud Sketch

You can edit the sketch by clicking the **sketch tab** inside your **Thing page**. The sketch is automatically generated with the enough code to upload it and connect to the cloud. 

Before uploading you should add the following code:

First include the headers that you need and declare the temperature sensor by adding the **temperature sensor ID**.

```cpp
#include "thingProperties.h"
#include "Arduino.h"
#include "Arduino_BHY2Host.h"

Sensor tempSensor(SENSOR_ID_TEMP);
```

***You can find all the Sensor IDs at <https://docs.arduino.cc/tutorials/nicla-sense-me/cheat-sheet#sensor-ids>***

Inside `void setup()` initialize the `Serial` communication, configure the variables and configuration for the Arduino IoT Cloud(properties), wait until the **Portenta H7** is connected to the Wi-Fi and IoT Cloud and after that it will setup the communication with the **Nicla Sense ME** and configure the temperature sensor.

***Note: Now we are using "NICLA_VIA_ESLOV". In case you mount it as a shield use "NICLA_AS_SHIELD" as the second parameter of the `begin()` function, or "NICLA_VIA_BLE" if you use Bluetooth® Low Energy***

```cpp
  void setup(){
    Serial.begin(9600);
    delay(1500);

    Serial.print("Configuring the Arduino IoT Cloud");
    // Defined in thingProperties.h
    initProperties();

    // Connect to Arduino IoT Cloud
    ArduinoCloud.begin(ArduinoIoTPreferredConnection);

    // Wait to be connected before intitalize the communication with the Nicla Sense ME
    Serial.println("Connecting to the Arduino IoT Cloud");
    while (ArduinoCloud.connected() != 1) {
      ArduinoCloud.update();
      delay(500);
    }

    delay(1500);

    Serial.println("Initialize the Nicla communication")
    BHY2Host.begin(false, NICLA_VIA_ESLOV);

    //If you want to connect the NICLA through Bluetooth® Low Energy use the following line instead of the above
    //while(!BHY2Host.begin(false, NICLA_VIA_BLE)) {} 

    tempSensor.configure(1, 0);
    temperature = tempSensor.value();
  }
```

***If you use `yourSensor.begin()` it will be configured the same as with `yourSensor.configure(1,0)`***

If the Nicla Sense ME shall communicate through Bluetooth® Low Energy, we recommend wrapping `BHY2Host.begin(false, NICLA_VIA_BLE)` in a `while` clause to make sure the connection is established before the sketch continues.

Inside the `void loop()` function you will make the **Portenta H7**  get all the needed data from the **Nicla Sense ME**, store and print the temperature sensor's value and update the data to the IoT Cloud.

```cpp
void loop(){
  BHY2Host.update();
  temperature = tempSensor.value();

  Serial.print("Temperature: ");
  Serial.println(temperature);

  ArduinoCloud.update();
}
```

Upload the sketch from the **sketch tab** by clicking the second button at the top left of the left side of the sketch bar. Once it has been uploaded you can see the temperature value that it is being uploaded by going to your **Thing Setup** tab, and looking at the last value of the **temperature**'s value, you can also open the serial monitor to see the live data.

![Arduino IoT Cloud - Sketch tab](assets/IoTCloud-thingSketch.png)
The Sketch:

```arduino
/*
  Sketch generated by the Arduino IoT Cloud Thing "PRO - Portenta and Nicla IoT"

  Arduino IoT Cloud Variables description

  The following variables are automatically generated and updated when changes are made to the Thing

  float temperature;

  Variables which are marked as READ/WRITE in the Cloud Thing will also have functions
  which are called when their values are changed from the Dashboard.
  These functions are generated with the Thing and added at the end of this sketch.
*/

#include "thingProperties.h"
#include "Arduino_BHY2Host.h"

Sensor tempSensor(SENSOR_ID_TEMP);

void setup() {
  Serial.begin(9600);
  delay(1500);

  Serial.print("Configuring the Arduino IoT Cloud");
  // Defined in thingProperties.h
  initProperties();

  // Connect to Arduino IoT Cloud
  ArduinoCloud.begin(ArduinoIoTPreferredConnection);

  Serial.println("Connecting to the Arduino IoT Cloud");
  while (ArduinoCloud.connected() != 1) {
    ArduinoCloud.update();
    delay(500);
  }

  delay(1500);
  

  Serial.println("Initialize the Nicla and the ");
  BHY2Host.begin(false, NICLA_VIA_ESLOV);
  tempSensor.configure(1, 0);
  temperature = tempSensor.value();
}

void loop() {
  // Your code here
  BHY2Host.update();
  temperature = tempSensor.value();
  Serial.print("Value: ");
  Serial.println(temperature);
  ArduinoCloud.update();

}

/*
  Since temperature is READ_WRITE variable, onTemperatureChange() is
  executed every time a new value is received from IoT Cloud.
*/
void onTemperatureChange()  {
  // Add your code here to act upon temperature change
}

```

## Conclusion

In this tutorial you learned how to upload the temperature values from the Nicla sense to the IoT cloud. You followed the process to set up the Nicla board to send data via the Eslov connection, and you configured the IoT cloud to receive the temperature data.

### Next Steps

- Try to upload other sensor data from he Nicla sense. You can see the available sensors in the [cheat sheet.](https://docs.arduino.cc/tutorials/nicla-sense-me/cheat-sheet#sensor-data-over-eslov)
- Experiment with the dashboard to add more data for a more sophisticated project.

## Troubleshooting

### ArduinoIoT Cloud
If you encounter any issue in the process of using the Arduino IoT Cloud, please visit the IoT Cloud Getting started <https://docs.arduino.cc/cloud/iot-cloud/tutorials/iot-cloud-getting-started>
