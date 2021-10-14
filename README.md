# Automation Workshop #5

This workshop allow students to understand how IoT services in Azure work and also will help them in their future projects of the signature.

This exercise require the student credits in Azure to create their own IoT Hub, connect the virtual device and pay attention to what happens next, view the logs sent by the device and so on.

# Getting Started

### Verify Credits

To verify Azure credits, do the following:

1. Log in into the Azure portal (https://portal.azure.com/#home) with your academic email.
2. Search for the the Education service in the Azure Services.
3. Go to the *Get started* option in the left bar.
4. If you have redeemed your credits, then you can continue with the Create IoT Hub section. If not, continue with the next point.
5. Activate your credits by Signing Up as a student.

## Create IoT Hub

To create your IoT Hub:

1. In the home page of Microsoft Azure portal, search for IoT Hub.
2. In the upper left corner of the IoT Hub, there is a *Create* button, click on it.
3. In Project Details, select Azure for Students as suscription and create your own resource group.
4. In the Instance Details, name as you want the IoT Hub name and leave East US.
5. Because it is a trial exercise, leave the Connectivity Configuration with Public Access.
6. In the Pricing and Scale Tier, select the F1: Free Tier and leave the Role-based access control as it is.
7. Finally, press the *Review + create* button.

## Create IoT Device

To create your IoT device:

1. Select the IoT Hub created in the Create IoT Hub section
2. In the middle left navigation bar, lookup in Explorers the IoT devices options
3. Select the *Add Device* button
4. Write down in the *Device ID* the name you want to put to your IoT Device
5. Select Save to create your IoT Device

# Workshop Development

First of all, enter to the Raspberry Pi Azure Web Simulator by clicking the following link: https://azure-samples.github.io/raspberry-pi-web-simulator/?lang=en#getstarted

Then, make sure you follow the steps in the Help section to connect the simulator to your IoT device create in the *Create IoT Device* section.

## Explanation
### Variables

```javascript

// wpi -> Access to the different Raspberry Pi devices
const wpi = require('wiring-pi');

// Client -> Send, receives and connect with Azure IoT Device client
const Client = require('azure-iot-device').Client;

// Message -> Send and receive messages from the IoT device to Azure IoT Device
const Message = require('azure-iot-device').Message;

// Protocol -> Connects the device with the IoT Device endpoints by MQTT protocols
const Protocol = require('azure-iot-device-mqtt').Mqtt;

// BME280 -> Bring all the sensor data
const BME280 = require('bme280-sensor');

// Object with optional variables 
const BME280_OPTION = {
  i2cBusNo: 1, // defaults to 1
  i2cAddress: BME280.BME280_DEFAULT_I2C_ADDRESS() // defaults to 0x77
};

// Primary Connection Key of the IoT device
const connectionString = 'HostName=Automation-class.azure-devices.net;DeviceId=First-device;SharedAccessKey=CJF6Za9KyxSiYJBRmajFrVA8y1nyCIDnPTKivMH/uk8=';
const LEDPin = 4;

// Initialize variables
var sendingMessage = false;
var messageId = 0;
var client, sensor;
var blinkLEDTimeout = null;
```
### Functions

```javascript
// Receives the message sent from the device and will return a JSON with the information of the message ID, the device temperature and humidity
function getMessage(cb) {
  messageId++;
  sensor.readSensorData()
    .then(function (data) {
      cb(JSON.stringify({
        messageId: messageId,
        deviceId: 'Raspberry Pi Web Client',
        temperature: data.temperature_C,
        humidity: data.humidity
      }), data.temperature_C > 30);
    })
    .catch(function (err) {
      console.error('Failed to read out sensor data: ' + err);
    });
}

// Send message if false adding temperatureAlert. If there is an error, it will send an error message to the IoT Hub, otherwise it will make the LED blink and will send a success message.
function sendMessage() {
  if (!sendingMessage) { return; }

  getMessage(function (content, temperatureAlert) {
    var message = new Message(content);
    message.properties.add('temperatureAlert', temperatureAlert.toString());
    console.log('Sending message: ' + content);
    client.sendEvent(message, function (err) {
      if (err) {
        console.error('Failed to send message to Azure IoT Hub');
      } else {
        blinkLED();
        console.log('Message sent to Azure IoT Hub');
      }
    });
  });
}

// Starts the process from obtaining the device information and different processes to send the information ti the IoT Hub
function onStart(request, response) {
  console.log('Try to invoke method start(' + request.payload + ')');
  sendingMessage = true;

  response.send(200, 'Successully start sending message to cloud', function (err) {
    if (err) {
      console.error('[IoT hub Client] Failed sending a method response:\n' + err.message);
    }
  });
}

// If true it will stop sending messages to the IoT Hub, otherwise it will send the the stop message with the error typo
function onStop(request, response) {
  console.log('Try to invoke method stop(' + request.payload + ')');
  sendingMessage = false;

  response.send(200, 'Successully stop sending message to cloud', function (err) {
    if (err) {
      console.error('[IoT hub Client] Failed sending a method response:\n' + err.message);
    }
  });
}

// Light up LED and sent the message by console if the message was sent successfully or not the data
function receiveMessageCallback(msg) {
  blinkLED();
  var message = msg.getData().toString('utf-8');
  client.complete(msg, function () {
    console.log('Receive message: ' + message);
  });
}

// It will turn the LED on/off everytime the system activates it o deactivates it
function blinkLED() {
  // Light up LED for 500 ms
  if(blinkLEDTimeout) {
       clearTimeout(blinkLEDTimeout);
   }
  wpi.digitalWrite(LEDPin, 1);
  blinkLEDTimeout = setTimeout(function () {
    wpi.digitalWrite(LEDPin, 0);
  }, 500);
}

// set up wiring
wpi.setup('wpi');
wpi.pinMode(LEDPin, wpi.OUTPUT);
sensor = new BME280(BME280_OPTION);
sensor.init()
  .then(function () {
    sendingMessage = true;
  })
  .catch(function (err) {
    console.error(err.message || err);
  });

// create a client
client = Client.fromConnectionString(connectionString, Protocol);

client.open(function (err) {
  if (err) {
    console.error('[IoT hub Client] Connect error: ' + err.message);
    return;
  }

  // set C2D and device method callback
  client.onDeviceMethod('start', onStart);
  client.onDeviceMethod('stop', onStop);
  client.on('message', receiveMessageCallback);
  setInterval(sendMessage, 2000);
});

```
