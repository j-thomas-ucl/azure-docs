---
title: Azure IoT Hub for Node.js getting started | Microsoft Docs
description: Azure IoT Hub with Node.js getting started tutorial. Use Azure IoT Hub and Node.js with the Microsoft Azure IoT SDKs to implement an Internet of Things solution.
services: iot-hub
documentationcenter: nodejs
author: dominicbetts
manager: timlt
editor: ''

ms.assetid: 56618522-9a31-42c6-94bf-55e2233b39ac
ms.service: iot-hub
ms.devlang: javascript
ms.topic: hero-article
ms.tgt_pltfrm: na
ms.workload: na
ms.date: 09/12/2016
ms.author: dobett

---
# Get started with Azure IoT Hub for Node.js
[!INCLUDE [iot-hub-selector-get-started](../../includes/iot-hub-selector-get-started.md)]

At the end of this tutorial, you have three Node.js console applications:

* **CreateDeviceIdentity.js**, which creates a device identity and associated security key to connect your simulated device.
* **ReadDeviceToCloudMessages.js**, which displays the telemetry sent by your simulated device.
* **SimulatedDevice.js**, which connects to your IoT hub with the device identity created earlier, and sends a telemetry message every second using the AMQP protocol.

> [!NOTE]
> The article [IoT Hub SDKs][lnk-hub-sdks] provides information about the various SDKs that you can use to build both applications to run on devices and your solution back end.
> 
> 

To complete this tutorial, you need the following:

* Node.js version 0.10.x or later.
* An active Azure account. (If you don't have an account, you can create a [free account][lnk-free-trial] in just a couple of minutes.)

[!INCLUDE [iot-hub-get-started-create-hub](../../includes/iot-hub-get-started-create-hub.md)]

You have now created your IoT hub. You have the IoT Hub hostname and the IoT Hub connection string that you need to complete the rest of this tutorial.

## Create a device identity
In this section, you create a Node.js console app that creates a device identity in the identity registry in your IoT hub. A device cannot connect to IoT hub unless it has an entry in the device identity registry. Refer to the **Device Identity Registry** section of the [IoT Hub Developer Guide][lnk-devguide-identity] for more information. When you run this console application, it generates a unique device ID and key that your device can use to identify itself when it sends device-to-cloud messages to IoT Hub.

1. Create a new empty folder called **createdeviceidentity**. In the **createdeviceidentity** folder, create a package.json file using the following command at your command-prompt. Accept all the defaults:
   
    ```
    npm init
    ```
2. At your command-prompt in the **createdeviceidentity** folder, run the following command to install the **azure-iothub** Service SDK package:
   
    ```
    npm install azure-iothub --save
    ```
3. Using a text editor, create a **CreateDeviceIdentity.js** file in the **createdeviceidentity** folder.
4. Add the following `require` statement at the start of the **CreateDeviceIdentity.js** file:
   
    ```
    'use strict';
   
    var iothub = require('azure-iothub');
    ```
5. Add the following code to the **CreateDeviceIdentity.js** file and replace the placeholder value with the connection string for the IoT hub you created in the previous section: 
   
    ```
    var connectionString = '{iothub connection string}';
   
    var registry = iothub.Registry.fromConnectionString(connectionString);
    ```
6. Add the following code to create a device definition in the device identity registry in your IoT hub. This code creates a device if the device id does not exist in the registry, otherwise it returns the key of the existing device:
   
    ```
    var device = new iothub.Device(null);
    device.deviceId = 'myFirstNodeDevice';
    registry.create(device, function(err, deviceInfo, res) {
      if (err) {
        registry.get(device.deviceId, printDeviceInfo);
      }
      if (deviceInfo) {
        printDeviceInfo(err, deviceInfo, res)
      }
    });
   
    function printDeviceInfo(err, deviceInfo, res) {
      if (deviceInfo) {
        console.log('Device id: ' + deviceInfo.deviceId);
        console.log('Device key: ' + deviceInfo.authentication.SymmetricKey.primaryKey);
      }
    }
    ```
7. Save and close **CreateDeviceIdentity.js** file.
8. To run the **createdeviceidentity** application, execute the following command at the command-prompt in the createdeviceidentity folder:
   
    ```
    node CreateDeviceIdentity.js 
    ```
9. Make a note of the **Device id** and **Device key**. You need these values later when you create an application that connects to IoT Hub as a device.

> [!NOTE]
> The IoT Hub identity registry only stores device identities to enable secure access to the hub. It stores device IDs and keys to use as security credentials and an enabled/disabled flag that you can use to disable access for an individual device. If your application needs to store other device-specific metadata, it should use an application-specific store. Refer to [IoT Hub Developer Guide][lnk-devguide-identity] for more information.
> 
> 

## Receive device-to-cloud messages
In this section, you create a Node.js console app that reads device-to-cloud messages from IoT Hub. An IoT hub exposes an [Event Hubs][lnk-event-hubs-overview]-compatible endpoint to enable you to read device-to-cloud messages. To keep things simple, this tutorial creates a basic reader that is not suitable for a high throughput deployment. The [Process device-to-cloud messages][lnk-process-d2c-tutorial] tutorial shows you how to process device-to-cloud messages at scale. The [Get Started with Event Hubs][lnk-eventhubs-tutorial] tutorial provides further information on how to process messages from Event Hubs and is applicable to the IoT Hub Event Hub-compatible endpoints.

> [!NOTE]
> The Event Hub-compatible endpoint for reading device-to-cloud messages always uses the AMQP protocol.
> 
> 

1. Create a new empty folder called **readdevicetocloudmessages**. In the **readdevicetocloudmessages** folder, create a package.json file using the following command at your command-prompt. Accept all the defaults:
   
    ```
    npm init
    ```
2. At your command-prompt in the **readdevicetocloudmessages** folder, run the following command to install the **azure-event-hubs** package:
   
    ```
    npm install azure-event-hubs --save
    ```
3. Using a text editor, create a **ReadDeviceToCloudMessages.js** file in the **readdevicetocloudmessages** folder.
4. Add the following `require` statements at the start of the **ReadDeviceToCloudMessages.js** file:
   
    ```
    'use strict';
   
    var EventHubClient = require('azure-event-hubs').Client;
    ```
5. Add the following variable declaration and replace the placeholder value with the connection string for your IoT hub:
   
    ```
    var connectionString = '{iothub connection string}';
    ```
6. Add the following two functions that print output to the console:
   
    ```
    var printError = function (err) {
      console.log(err.message);
    };
   
    var printMessage = function (message) {
      console.log('Message received: ');
      console.log(JSON.stringify(message.body));
      console.log('');
    };
    ```
7. Add the following code to create the **EventHubClient**, open the connection to your IoT Hub, and create a receiver for each partition. This application uses a filter when it creates a receiver so that the receiver only reads messages sent to IoT Hub after the receiver starts running. This filter is useful in a test environment so you see just the current set of messages. In a production environment, your code should make sure that it processes all the messages - see the [How to process IoT Hub device-to-cloud messages][lnk-process-d2c-tutorial] tutorial for more information:
   
    ```
    var client = EventHubClient.fromConnectionString(connectionString);
    client.open()
        .then(client.getPartitionIds.bind(client))
        .then(function (partitionIds) {
            return partitionIds.map(function (partitionId) {
                return client.createReceiver('$Default', partitionId, { 'startAfterTime' : Date.now()}).then(function(receiver) {
                    console.log('Created partition receiver: ' + partitionId)
                    receiver.on('errorReceived', printError);
                    receiver.on('message', printMessage);
                });
            });
        })
        .catch(printError);
    ```
8. Save and close the **ReadDeviceToCloudMessages.js** file.

## Create a simulated device app
In this section, you create a Node.js console app that simulates a device that sends device-to-cloud messages to an IoT hub.

1. Create a new empty folder called **simulateddevice**. In the **simulateddevice** folder, create a package.json file using the following command at your command-prompt. Accept all the defaults:
   
    ```
    npm init
    ```
2. At your command-prompt in the **simulateddevice** folder, run the following command to install the **azure-iot-device** Device SDK package and **azure-iot-device-amqp** package:
   
    ```
    npm install azure-iot-device azure-iot-device-amqp --save
    ```
3. Using a text editor, create a new **SimulatedDevice.js** file in the **simulateddevice** folder.
4. Add the following `require` statements at the start of the **SimulatedDevice.js** file:
   
    ```
    'use strict';
   
    var clientFromConnectionString = require('azure-iot-device-amqp').clientFromConnectionString;
    var Message = require('azure-iot-device').Message;
    ```
5. Add a **connectionString** variable and use it to create a device client. Replace **{youriothostname}** with the name of the IoT hub you created the *Create an IoT Hub* section. Replace **{yourdevicekey}** with the device key value you generated in the *Create a device identity* section:
   
    ```
    var connectionString = 'HostName={youriothostname};DeviceId=myFirstNodeDevice;SharedAccessKey={yourdevicekey}';
   
    var client = clientFromConnectionString(connectionString);
    ```
6. Add the following function to display output from the application:
   
    ```
    function printResultFor(op) {
      return function printResult(err, res) {
        if (err) console.log(op + ' error: ' + err.toString());
        if (res) console.log(op + ' status: ' + res.constructor.name);
      };
    }
    ```
7. Create a callback and use the **setInterval** function to send a new message to your IoT hub every second:
   
    ```
    var connectCallback = function (err) {
      if (err) {
        console.log('Could not connect: ' + err);
      } else {
        console.log('Client connected');
   
        // Create a message and send it to the IoT Hub every second
        setInterval(function(){
            var windSpeed = 10 + (Math.random() * 4);
            var data = JSON.stringify({ deviceId: 'myFirstNodeDevice', windSpeed: windSpeed });
            var message = new Message(data);
            console.log("Sending message: " + message.getData());
            client.sendEvent(message, printResultFor('send'));
        }, 1000);
      }
    };
    ```
8. Open the connection to your IoT Hub and start sending messages:
   
    ```
    client.open(connectCallback);
    ```
9. Save and close the **SimulatedDevice.js** file.

> [!NOTE]
> To keep things simple, this tutorial does not implement any retry policy. In production code, you should implement retry policies (such as an exponential backoff), as suggested in the MSDN article [Transient Fault Handling][lnk-transient-faults].
> 
> 

## Run the applications
You are now ready to run the applications.

1. At a command-prompt in the **readdevicetocloudmessages** folder, run the following command to begin monitoring your IoT hub:
   
    ```
    node ReadDeviceToCloudMessages.js 
    ```
   
    ![Node.js IoT Hub service client application to monitor device-to-cloud messages][7]
2. At a command-prompt in the **simulateddevice** folder, run the following command to begin sending telemetry data to your IoT hub:
   
    ```
    node SimulatedDevice.js
    ```
   
    ![Node.js IoT Hub device client application to send device-to-cloud messages][8]
3. The **Usage** tile in the [Azure portal][lnk-portal] shows the number of messages sent to the hub:
   
    ![Azure portal Usage tile showing number of messages sent to IoT Hub][43]

## Next steps
In this tutorial, you configured a new IoT hub in the Azure portal, and then created a device identity in the hub's identity registry. You used this device identity to enable the simulated device app to send device-to-cloud messages to the hub. You also created an app that displays the messages received by the hub. 

To continue getting started with IoT Hub and to explore other IoT scenarios, see:

* [Connecting your device][lnk-connect-device]
* [Getting started with device management][lnk-device-management]
* [Getting started with the IoT Gateway SDK][lnk-gateway-SDK]

To learn how to extend your IoT solution and process device-to-cloud messages at scale, see the [Process device-to-cloud messages][lnk-process-d2c-tutorial] tutorial.

<!-- Images. -->
[7]: ./media/iot-hub-node-node-getstarted/runapp1.png
[8]: ./media/iot-hub-node-node-getstarted/runapp2.png
[43]: ./media/iot-hub-csharp-csharp-getstarted/usage.png

<!-- Links -->
[lnk-transient-faults]: https://msdn.microsoft.com/library/hh680901(v=pandp.50).aspx

[lnk-eventhubs-tutorial]: ../event-hubs/event-hubs-csharp-ephcs-getstarted.md
[lnk-devguide-identity]: iot-hub-devguide-identity-registry.md
[lnk-event-hubs-overview]: ../event-hubs/event-hubs-overview.md

[lnk-dev-setup]: https://github.com/Azure/azure-iot-sdks/blob/master/doc/get_started/node-devbox-setup.md
[lnk-process-d2c-tutorial]: iot-hub-csharp-csharp-process-d2c.md

[lnk-hub-sdks]: iot-hub-devguide-sdks.md
[lnk-free-trial]: http://azure.microsoft.com/pricing/free-trial/
[lnk-portal]: https://portal.azure.com/

[lnk-device-management]: iot-hub-device-management-get-started.md
[lnk-gateway-SDK]: iot-hub-linux-gateway-sdk-get-started.md
[lnk-connect-device]: https://azure.microsoft.com/develop/iot/
