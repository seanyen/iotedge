EdgeHub mirrors IoT Hub operations and gives support for the following:
- Send Telemetry messages
- Get Twin
- Get Desired-Properties Update
- Send Reported-Properties
- Receive Cloud-to-Device messages
- Receive Direct-Method calls

## Telemetry Message
Telemetry messages, or sometimes called device-to-cloud messages, carry custom data from devices/modules to IoT Hub/Edge. They have a user defined payload, some system properties and optional user properties. 

The structure of the message and the type of system properties can be added is described on the [IoT Hub documentation](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-messages-construct).
Telemetry messages are subject to [Routing](https://docs.microsoft.com/en-us/azure/iot-edge/module-composition?view=iotedge-2020-11#declare-routes). When telemetry messages are routed to an [Edge Module](https://docs.microsoft.com/en-us/azure/iot-edge/module-development?view=iotedge-2020-11), we are talking about Module-to-Module messages.

If Edge Hub is between IoT Hub and a device/module, Edge Hub acts as a buffer and router. It does not just relay the messages. When a device/module sends a message, Edge Hub takes it and acknowledges it even [if it cannot forward the message to IoT Hub](https://docs.microsoft.com/en-us/azure/iot-edge/offline-capabilities) or to another module immediately. Depending on the settings, Edge Hub stores the message in a persistent store (which is the default behavior) or holds it in memory. 

After receiving a message, Edge Hub runs the message through its routing, and depending on the routing rules Edge Hub tries to forward the message to IoT Hub or another module. If it is not possible, because Edge Hub is not in connection to IoT Hub, or the module is off, then the message stays stored and will be delivered next time it is possible.
## Twin
[Twins](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-device-twins) are basically a combination of configuration and status information. To understand the term "Twin" and the role of this construct, let's say that we want to control a beverage cooler and we want its temperature to set 5 Celsius.

In this case the part of the twin that describes the settings/configuration we want, can look the following:
```
"properties": {
    "desired": {
        "temperature": 5
    }
}
```
After the setting has been made, it might be useful to know how the device - in this case the beverage cooler - progresses achieving the desired state. To support this scenario, twins have another section where devices can report their status:
```
"properties": {
    "desired": {
        "temperature": 5
    },
    "reported": {
        "temperature": 7
    }
}
```
The device can periodically report its status and many times the desired section and the reported section converges and finally take the same values. Hence the name Twin.

The "desired" section is usually set by a backend application, or the operator can use Azure Portal to set the twin. The "reported" part is usually generated by the device/module.

Note, that it is not necessary that the "desired" and "reported" sections have the same properties. It is possible that the "desired" section contains a property which later is not going to be reported back. Also, it can happen that a device reports status information that does not have a corresponding part in the "desired" section.

From the perspective of a device/module, so far two kind of operations were mentioned:
- reading the desired properties section
- writing the reported properties section

Iot Hub and Edge Hub allow two ways to read the desired properties:
- Pulling the entire twin (including the desired properties section)
- Receiving patches for desired property changes

The first way - pulling the entire twin - is usually the initial step when a device/module connects to Iot Hub/Edge Hub. This allows processing the entire desired properties section, which might have changed while the device was offline.
Later, during normal operation, the device/module is probably interested only in changes of the desired properties, and the second method allows to receive only those smaller updates.

When a device/module signed up to receive the desired propery updates, it receives blocks similar to the following:

```
{ "some-property": 123 }
```

If only "some-property" has been changed recently, that is all the device is going to receive, even if the desired properties section has 100 other properties. These are called patches and they allow to minimize band width usage.

There is one more important concept using Twins. Both the desired and the reported part has a version number maintained by IoT Hub. This version number changes every time the corresponding part - the desired or reported part - changes.

Let's say that a device has just connected to Iot Hub/Edge Hub and it pulled the entire twin. It receives something like the following:

```
"properties": {
    "desired": {
        "temperature": 5,
        $version: 23
    },
    "reported": {
        "temperature": 7
        $version: 98
    }
}
```

Compared to the earlier example, this desired properties section has a $version field with a numeric value. The above mentioned patch also always have a version number included:

```
{ "some-property": 123, "$version" : 25 }
```

Comparing the result of the twin pull operation and the desired property patch, one has the version 23 and the other 25. That means that between the twin pull and the patch there might have been another change.

To avoid missing patches, the recommended way of using twins is the following:
- Subscribe for desired property changes
- Pull the entire twin, parse the desired property changes section and store the $version
- Going forward check the $version propery of every incoming patch. If the version number incremented by more then one, there are odds that a patch was missed, so pull the entire twin.

Edge Hub - like in case of other operations - hides disconnections from connected devices/modules. If Edge Hub got disconnected from IoT Hub for any reason, after reconnection it pulls the twins for those connected modules/devices that have twin subscriptions. Then it checks the content of the twins with its own stored replicas. When Edge Hub detects any changes, it generates a patch and sends it to the devices/modules.

## Cloud-to-Device messages
Twin is a simple way to send configuration to a device or module, but it is not feasible to send complex information or data with bigger size. If that is a requirement, [cloud-to-device](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-messages-c2d) or C2D messages can be used to send arbitrary data, however these can be sent only to devices, not modules.

C2D messages are much like telemetry messages. A back-end application pushes a message to IoT Hub. IoT Hub enqueues that message and stores it (up to 48 hours). If the device is online, Iot Hub sends the message to the device and waits for a feedback. 

This feedback can be 'completing' the message, 'rejecting' or 'Abandoning' it. Completing the message means that it was processed succesfully and no need for the message anymore, so IoT Hub deletes it. Rejecting the message means that the device does not want to process the message. In this case IoT Hub moves it in a dead letter queue where it is available for a while for a back end application. Abandoning means that the message cannot be processed right now. In this case IoT Hub puts it back the the queue and tries again later.

If Edge Hub is between the device and Iot Hub, Edge Hub relays the messages and the feedbacks. It does not 'download' the C2D messages so it can forward them later. When the device signals Edge Hub that it is ready to receive C2D messages, Edge Hub signals in the name of the device that it is ready to receive C2D messages. When IoT Hub sends a C2D message, then Edge Hub sends it to the device and waits for the feedback. When the feedback arrives, Edge Hub propagates it back to IoT Hub.

## Direct-Method calls

Twins and C2D messages can deliver data to a device or module, but they are limited in terms of providing response data. Twin reported properies are a way to deliver some response, but it is not designed to be used as a method call, when there is an action to perform and an immediate response to that request. That is where [Direct-Method](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-direct-methods) calls can be useful.

A direct method call can deliver a payload, which is in json format and can be up to 128KB. Then the device/module needs to respond immediately (meaning between a configurable 5-300 second interval). The response also can deliver some data up to 128KB.

Just like in the case of C2D messages, Edge Hub relays the messaging between IoT Hub and the device/module.

## Edge Hub support for IoT Hub operations
Edge Hub emulates IoT Hub for connecting devices/modules. It provides the same API, but extends IoT Hub functionality with local message routing and some offline features.

Local routing was introduced to made it possible to process telemetry data locally by specialized modules. This feature can be used to save bandwidth to IoT Hub or provide faster feedback to the data.

The offline features include buffering data from devices/modules, collecting reported properties, or provide the latest known twin for devices/modules when they become online, but there is no connection to IoT Hub.

EdgeHub support AMQP and MQTT endpoints, but not HTTPS. It opens an HTTPS port but only to support websockets for AMQP and MQTT **[FIXME there is something else on that https]**

## Edge Hub MQTT Endpoint
Edge Hub replicates [IoT Hub support for MQTT](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-mqtt-support). It allows connecting on port 8883 or port 443 with WebSocket. Both ports requires TLS to be used.
### Connecting
After a TLS connection is established, the very first packet to be sent is the MQTT Connect packet. A connect packet has three important fields to deliver:
- Client Id
- Username
- Password

Client Id has the following form:
- for leaf devices it is the device id, e.g: TestDevice
- for modules it is in the form of: EdgeDeviceId/ModuleId

The username:
- for leaf devices it is in the form of: your-hub-name.azure-devices.net/DeviceId/?api-version=2018-06-30
- for modules: your-hub-name.azure-devices.net/EdgeDeviceId/ModuleId/?api-version=2018-06-30

The password field is needed only for SAS authentication. In case of certificate authentication the TLS handshake delivers the client certificate, which Edge Hub uses for authentication. For certificate authentication the password field must be omitted from the mqtt connect packet.

The password field is a generated SAS token in the following form: 
```
SharedAccessSignature sig={signature-string}&se={expiry}
```
For example using an IoT Hub named 'test-srv' and a device named 'test_device':
```
SharedAccessSignature test-srv.azure-devices.net%2Fdevices%2Ftest_device&sig=Mzv6sxyYH6WWfCpnYtkQBaF0IwfTPJFOULfWB80BEvQ%3D&se=1637291060
```

The SAS token for a device or module can be generated several different ways:
- Using Visual Studio Code with [Azure IoT Hub extension](https://marketplace.visualstudio.com/items?itemName=vsciot-vscode.azure-iot-toolkit), using [these steps](https://github.com/Microsoft/vscode-azure-iot-toolkit/wiki/Generate-SAS-Token-for-IoT-Hub)
- Using [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) with [IoT Extension](https://github.com/Azure/azure-iot-cli-extension) a [generate-sas-token](https://docs.microsoft.com/en-us/cli/azure/iot/hub?view=azure-cli-latest#az_iot_hub_generate_sas_token) command is available
- The token generation can be implemented by [program code](https://github.com/microsoft/azure-docs/blob/master/articles/iot-hub/iot-hub-dev-guide-sas.md)

If there is an MQTT client library where an MQTT client object has a Connect() method with the signature:
```
mqttClient.Connect(string clientId, string username, string password)
```

Then using this method to connect a device named 'TestDevice' using an IoT Hub named 'test-srv', a possible call is:

```
mqttClient.Connect(
                "TestDevice",
                "test-srv.azure-devices.net/TestDevice?api-version=2018-06-30",
                "SharedAccessSignature sr=test-srv.azure-devices.net%2Fdevices%2FTestDevice&sig=Mzv6sxyYH6WWfCpnYtkQBaF0IwfTPJFOULfWB80BEvQ%3D&se=1637291060")
```
Edge Hub supports the Clean Session flag in the MQTT packet. If it is not set and a device or module had a session previously, then Edge Hub restores the previous subscriptions (e.g. for receiving twin patches)
### Sending Telemetry
Sending telemetry message is the most common and simplest operation. Edge Hub supports sending using Qos0 and Qos1. In order to send telemetry:
- devices can use the topic: devices/{device_id}/messages/events
- modules can use the topic: devices/{device_id}/modules/{module_id}/messages/events, where device_id is the name of the Edge device

Telemetry messages can encode properties similarly than URL query strings. This section is called 'property bag' and must be encoded according to [RFC2396](https://www.ietf.org/rfc/rfc2396.txt). The following example adds two parameters to the telemetry message; the first parameter name is called "food" and its value is "I like fries" and the second parameter name is "pet" and its value is "I like cats":
```
devices/test_device/messages/events/?food=I%20like%20fries&pet=I%20like%20cats
```
The properties can be retrieved as user properties by the back end application. Note, that Edge Hub/IoT Hub may add additional system properties to the message.

When Edge Hub receives a telemetry message and it is QoS1, it acknowledges the message immediately. It does not mean that the message will be forwarded right away. Edge Hub stores the message, it runs the message through its [routing](https://docs.microsoft.com/en-us/azure/iot-edge/module-composition?view=iotedge-2020-11#declare-routes) and then it decides what to do. If Edge Hub is currently connected to IoT Hub and the message is routed upstream, then the message is going to be forwarded upstream. Otherwise the message may be sitting in the internal store of Edge Hub until the target becomes available. It is also possible that the routing does not target any device for the message and it gets dropped.
### Receiving Module-to-Module messages
A telemetry message sent by a device or module can be routed to another module. If a module wants to receive M2M messages, first it needs to subscribe to the topic which delivers them. The format of the subscription is:
```
devices/{device_id}/modules/{module_id}/#
```
If there is an Edge device called TestEdgeDevice and a module named TestModule wants to receive M2M messages, its subscription would be:
```
devices/TestEdgeDevice/modules/TestModule/#
```
Depending on the routing settings, the routing may define an [input name](https://docs.microsoft.com/en-us/azure/iot-edge/module-composition?view=iotedge-2020-11#sink), which will be attached to the topic when a message is getting forwarded. Also, Edge Hub (and the original sender) adds parameters to the message which is encoded in the topic structure. The following example shows a message routed with input name "TestInput". This message was sent by a module called "SenderModule", which name is also encoded in the topic:
```
devices/TestEdgeDevice/modules/TestModule/inputs/TestInput/%24.cdid=TestEdgeDevice&%24.cmid=SenderModule
```
Modules can also send messages on a specific [output name](https://docs.microsoft.com/en-us/azure/iot-edge/module-composition?view=iotedge-2020-11#source). Output names help when messages from a module need to be routed to different destinations. When a module wants to send a message on a specific output, it sends the message as an regular telemetry message, except that it adds an additional system property to it. This system property is '\$.on'. The '\$' sign needs to be url encoded and it becomes %24 in the topic name. The following example shows a telemetry message sent with the output name 'alert':
```
devices/TestEdgeDevice/modules/TestModule/messages/events/%24.on=alert/
```
### Getting Full Twin
Pulling the twin from IoT Hub/Edge Hub has a request/response pattern using MQTT. The request is a publication to a certain topic, where the topic name contains a Request Id encoded in it. When IoT Hub/Edge Hub provides the requested twin, it publishes it on a predefined topic, attaching the request id to the topic name.

Because of the request/response pattern, IoT Hub/Edge Hub publishes the twin on a topic and the device or module that requested the twin needs to be subscribed to this topic to receive the result. Specificly, the device/module needs to subscribe to the following topic:
```
$iothub/twin/res/#
```
When a device/module wants to get the latest twin, it needs to generate a request id, which is an arbitrary string, like a guid or an incremented counter. Then it needs to publish an empty message on the following topic:
```
$iothub/twin/GET/?$rid={request id}
```
When Iot Hub/Edge Hub responds to the twin request, it sends back 3 things:
- a status code for the operation, which is a similar number to HTTP status codes
- the request id the client used to request the twin
- the twin itself

The status code is usually 200, which means successful operation. The code 429 is returned in case of too many twin request, when IoT Hub throttled the request. Status codes starting with 5** are other errors.

An example response topic for successful operation and request id = 1234:
```
$iothub/twin/res/200/?$rid=1234
```
The mqtt message payload is an utf8 encoded json document showed earlier describing twin messages,

When using Edge Hub, the twin request will be handled the following way:
- When a device/module connects to Edge Hub, it pulls the latest twin from IoT Hub, so it has a copy
- When a device/module sends a twin request, Edge Hub forwards the request to Iot Hub and waits for the answer
- When a response arrives from Iot Hub, Edge Hub relays the answer to the device/module.
- If there is an error requesting the twin from IoT Hub, Edge Hub sends its latest copy to the device/module instead.
### Sending Reported Property Updates
Reported Property Updates are request/response pairs similarly to twin pulls. The request/response is necessary because the update can fail for different reason and the sender needs to know to repeat it.

Just like for twin pulls, the sender needs to generate a request id, which will be sent back by IoT Hub/Edge Hub. Then the update message must be published on the following topic:
```
 $iothub/twin/PATCH/properties/reported/?$rid={request id}
```
The payload of the update message describes all the properties with the new values:
```
{
    "UpdatedProperty1" : "new value",
    "UpdatedProperty2" : 123,
    "UpdatedProperty3" : null
}
```
In the example three properties are getting updated, all the rest (if any) of the reported properies will keep their old values. The third "UpdatedProperty3" makes the propery deleted, as it is described in the [documentation of device twins](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-device-twins).

The response is going to be published on a topic like the following:
```
$iothub/twin/res/204/?$rid=1234
```

This also means that a device/module needs to subscribe to "$iothub/twin/res/#" before sending a patch, otherwise it would never learn about the result of the operation. Normally IoT Hub/Edge Hub responds 204, which is the HTTP OK with no content. Error code 400 means malformed json, 429 throttling and 5** a server error.

In case of reported properties, Edge Hub intercepts the message and reponds it by itself. Later (which normally means immediately) it synchronizes the patches with IoT Hub. If Edge Hub is offline, then it accumulates the patches and sends to IoT Hub when it gets online.

### Receiving Desired Property Updates
In order to receive desired property updates, a device/module needs to subscribe to a topic where these updates get published. This topic is:
```
$iothub/twin/PATCH/properties/desired/#
```
When the desired properties section of the twin gets changed in IoT Hub, it sends a json document in the same format as reported properties updates get sent.

Edge Hub only relays desired propery changes: when a device/module connects to Edge Hub and subscribes to desired property changes, Edge Hub forwards this subscription to IoT Hub. When IoT Hub sends a patch, then the patch will be forwarded to the device/module. If the device/module gets disconnected, then Edge Hub will no longer receives patches from Iot Hub. It does not work the way that until a device is not connected, Edge Hub keeps gathering the patches and sends it out once the device comes back. If a device/module gets disconnected, it always needs to pull its twin to have the latest version.

### Receiving Cloud-to-Device messages
Just like other message types, C2D messages cannot be received until a device subscribes to them. C2D messages cannot be sent to modules, only to devices. When a device is ready to receive C2D messages, it subscribes to the following topic:
```
devices/{device_id}/messages/devicebound/#
```
When IoT Hub sends a message, the it will be published on the topic similar to the following:
```
devices/dev_c/messages/devicebound/propertyName=propertyValue&%24.to=%2Fdevices%2Fdev_c%2Fmessages%2FdeviceBound
```
In this example the device name is 'dev_c'. The topic encodes a user property called 'propertyName', which has the value 'propertyValue'. Also, there is a system propert added to the topic ('$.to').

The response to a C2D message at application level can be 'complete', 'reject' and 'abandon', however the MQTT protocol is not able to express all of these. When a C2D message is received by a device and the device acknowledges the MQTT packet that delivered the C2D message, Edge Hub takes the acknowledgement as 'complete' and forwards this back to IoT Hub. If the receiving client does not acknowledges the MQTT packet, IoT Hub is going to be timeout after while and it takes it as 'abandon'.

### Receiving Direct-Method calls

Similarly to C2D messages and twins, Direct-Method calls need to be subscribed to. IoT Hub and Edge Hub publishes direct method calls on a topic:
```
$iothub/methods/POST/{method name}/?$rid={request id}
``` 
According to that publication, a client that wants to receive direct method calls needs to subsribe to:
```
$iothub/methods/POST/#
``` 
When IoT Hub sends a direct method call, Edge Hub relays that call and publishes an MQTT message like the following:
```
$iothub/methods/POST/some_method/?$rid=505e09bb-0076-4b9f-b4a3-529430f1593f
```
This call has the method name "some_method" and a guid as request id. This request id must be sent back with the response payload.
The payload of the direct method call can be an arbitrary json document:
```
{
    "some_property" : "some_value",
    "some_other_property" : 123
}
```
When the client processed the direct method call, it can send back a payload and also a status code as a result of a call. This status code can be any number that the caller application can interpret. Also, the payload can be any json document. An example response to the direct method call earlier can be published to the following topic:
```
$iothub/methods/res/200/?$rid=505e09bb-0076-4b9f-b4a3-529430f1593f
```
There is a configurable time limit the device/module needs to answer, which is 30 seconds by default, but can be set from 5 to 300 seconds.
## Edge Hub MQTT Endpoint Extensions
Edge Hub runs a full-fledged MQTT broker. **This is an experimental feature** and [needs to be turned on in order to use](https://docs.microsoft.com/en-us/azure/iot-edge/how-to-publish-subscribe?view=iotedge-2020-11#prerequisites). Also, **the MQTT broker related extensions are subject to change.**

If the MQTT Broker experimental feature is turned on, then the operations discussed above can be also accessed by a different set of MQTT topics. One limitation of the existing topic structure is that many operations do not contain device/module id. IoT Hub and Edge Hub know from the connection which device needs to receive a message and they send directly to that specific device, regardless of the subscription of other devices.

If there are two devices, device_a and device_b, and both subscribed to $iothub/twin/res/#, then device_a requests a twin, it will not be forwarded to device_b, even if the response is published on a topic starting with $iothub/twin/res/, and device_b is subscribed to that topic.

Edge Hub MQTT extension uses a set of topics which includes device/module ids. This allows some scenarios not possible with legacy topics and can be useful for identity translation. Twins can be requested for example on a topic strucure '$iothub/{device_id}/twin/get/?$rid=1234'. Let's say that there are 5 different devices that too simple to be able to programmable to communicate using IoT Hub protocols. In this case it is still prossible to write a module that takes over IoT Hub communication and it is able to request the twin of any of those devices just changing the device_id part of the twin request topic - given that the authorization settings allow to do that.

### Telemetry messages
In case of devices, a telemetry message can be sent to Edge Hub if the MQTT broker is turned on using the following topics:

If the client is a device:
```
$iothub/{device_id}/messages/events
```
If the client is a module:
```
$iothub/{device_id}/{module_id}/messages/events
```
Other parameters can be encoded into the topic similarly to the legacy telemetry topics.
### Getting Full Twin
If a client wants to be able to receive the result of its twin request, then it needs to subsribe to the following topic:

If the client is a device:
```
$iothub/{device_id}/twin/res/#
```
If the client is a module:
```
$iothub/{device_id}/{module_id}/twin/res/#
```
After the subscription, the client can request a twin by publishing an empty message on the topic:

If the client is a device:
```
$iothub/{device_id}/twin/get/?$rid={request_id}
```
If the client is a module:
```
$iothub/{device_id}/{module_id}/twin/get/?$rid={request_id}
```
Edge Hub is going to send the twin publishing on a topic similar to the following:
```
$iothub/some_device/some_module/twin/res/200/?$rid=1234
```
### Sending Reported Property Updates
For sending propery updates, the client first needs to subscribe to the twin response topic just like when it pulls the full twin. After the subscription, the client can publish its update on the following topic:

If the client is a device:
```
$iothub/{device_id}/twin/reported/?$rid={request_id}
```
If the client is a module:
```
$iothub/{device_id}/{module_id}/twin/reported/?$rid={request_id}
```
### Receiving Desired Property Updates
The topic to subscribe for desired properties,

If the client is a device:
```
$iothub/{device_id}/twin/desired/#
```
If the client is a module:
```
$iothub/{device_id}/{module_id}/twin/desired/#
```
### Receiving Cloud-to-Device messages
In order to receive C2D messages, the client needs to subscribe to the following topic:
```
$iothub/{device_id}/messages/c2d/post/#
```
### Receiving Direct-Method calls
To be able to receive direct method calls, a client needs to subscribe to:

If the client is a device:
```
$iothub/{device_id}/methods/post/#
```
If the client is a module:
```
$iothub/{device_id}/{module_id}/methods/post/#
```

Edge Hub is going to send a direct method call similarly to the following:
```
$iothub/some_device/some_module/methods/post/some_method/?$rid=1234
```
In response the client needs to publish on the following topic:

```
$iothub/some_device/some_module/methods/res/200/?$rid=1234
```
## Edge Hub HTTPS Endpoint
Edge Hub opens up port 443 for HTTPS communication. This port serves three different public operations:
- Amqp communication over WebSocket
- MQTT communication over WebSocket
- Direct Method calls

The WebSocket communication is based on [rfc6455](https://datatracker.ietf.org/doc/html/rfc6455). After the connecting client established an https connection and played the WebSocket handshake described in the RFC, Edge Hub routes the traffic to the appropriate (Ampq or Mqtt) protocol head.

### Direct Method Call
In case of Direct Method calls, Edge Hub emulates the IoT [Hub functionality](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-direct-methods) with some restrictions: the caller can be only a module from the edge device. The specification of the request/response is the following:

#### Request
```
POST /twins/{device-id}/methods?api-version={version}

Headers:
    x-ms-edge-moduleId : {caller module-id}
    Authorization : {generated sas-token of the caller}
    Content-Type : application/json

Request Body:
    {
        "methodName": {method-name to be called},
        "responseTimeoutInSeconds": {integer value},
        "payload": json-object
    }
```

#### Response
```
Response Codes:
    200 - Successful call
    404 - Invalid devic-id or the device is offline
    504 - Timeout

Response Body:
    {
        "status" : {integer value sent by the called device},
        "payload" : json-object
    }

```
#### Example
To call the endpoint, a module first needs to generate a SAS token for the caller. The following example shows it using [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) with [IoT Extension](https://github.com/Azure/azure-iot-cli-extension):

```
az iot hub generate-sas-token -n test-srv -d TestEdgeDevice -m TestCallerModule --key-type primary --du 360000
```

Using the SAS token, the following shows a call to the 443 port for direct methods:

```
curl -X POST 
   https://localhost/twins/dev_c/methods?api-version=2018-06-30 
   -H "x-ms-edge-moduleId":"TestEdgeDevice/TestCallerModule" 
   -H 'Authorization: SharedAccessSignature sr=test-srv.azure-devices.net%2Fdevices%2FTestEdgeDevice%2Fmodules%2FTestCallerModule&sig=tm6Qwy9b3qWiKNChddzdfYSu%2BtDoYh%2Bq7hMqXlOavkk%3D&se=1637558448' 
   -H 'Content-Type: application/json' 
   -d '{"methodName": "TestMethod",
        "responseTimeoutInSeconds": 200,
        "payload": {"some_parameter": "some_value", "other_parameter": 123}}'

```

The payload of the direct method call is different to that when the call is made e.g. from azure portal: it needs to specify the method name to be called and the response timeout.

The response of the call is also an extended json document compared to when only the payload travells: it contains a status information with HTTP status codes. The response to the call above can be like the following:

```
{ 
    "status": 200,
    "payload":  {
                    "some_result_name" : "some_value"
                }
}
```