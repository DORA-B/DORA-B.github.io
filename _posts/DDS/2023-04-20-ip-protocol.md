SOME/IP: Scalabeservice-Oriented Mideleware over IP, IP-based extensible service-oriented middleware.

In the protocol architecture of Ethernet in vehicles, SOME/IP is in the application layer, Provides a service-oriented communication interface.
The communication method is the concept of the CS interface mentioned in AUTOSAR, which refers to the client and server. When a request is made, SOME/IP sends out data; otherwise, no data is sent, similar to the Direct mode of the COM module. This reduces unnecessary data on the bus and lowers the bus load.

At the same time, Classical AUTOSAR and Adaptive AUTOSAR can be bridged together in the overall vehicle electronic and electrical architecture.
![SOME/IP in AUTOSAR](/commons/images/3141911-20230413102015556-1559946077.png#pic_center)
   
![Automotive Ethernet protocol architecture](/commons/images/3141911-20230413102359397-66928387.png)

![Bridging between CP and AP, the way described before](/commons/images/3141911-20230413102511237-136910100.png)

SOME/IP mainly provides API interfaces for the application layer, creates CS interfaces, and communicates through TCP/IP protocol. The access methods of SOME/IP are divided into three types: event notification, remote procedure call, and access to process data.

1. Event notification is similar to traditional CAN communication, where the server periodically or upon event changes sends specific data to the client.
![event notification](/commons/images/3141911-20230413102734183-69730732.png)

2. Remote procedure call is when the client sends a request command to the server when there is a request, the server parses the command and responds accordingly.
![remote procedure call](/commons/images/3141911-20230413102750285-1335933944.png)

3. Accessing process data allows clients to write (Setter) or read (Getter) data to the server.

![access process data](/commons/images/3141911-20230413102931300-1976098639.png)

The data format of SOME/IP:
![data format of SOME/IP](/commons/images/3141911-20230413103054141-1553358951.png)

Message ID (Server ID): 16bit, service ID, identifies a service;

Message ID (Method ID): 16bit, the ID of the method, indicating a method;

Length: message length, 32bit, identifies the total length from the request ID to the end of the message;

Request ID (Client ID): client ID, 16bit. Distinguish between different clients;

Request ID (Session ID): Session ID, which distinguishes multiple calls from the same client;

Protocol Version: the version number of the protocol, the fixed value is x01;

Interface Version: service interface version;

Message Type: message type, in AUTOSAR, contains five types in total, including REQUEST, REQUEST_NO_RETURN, NOTIFICATION, RESPONSE, ERROR;

Return Code: return code, including four types, REQUEST, REQUEST_NO_RETURN, NOTIFICATION, RESPONSE;

Payload: The data segment, used to place the data that needs to be transmitted.



*** Compare between SOME/IP and DDS ***

|features|SOME/IP|DDS|
|:---|:---:|:---:|
|Communication Model|Request/Response + Subscribe/Publish|Subscribe/Publish|
|Architectural Style|Service-Oriented|Data-Centric|
|Transport Protocol|TCP/UDP|UDP(default)/TCP/Shared-Memory|
|Dynamic Discovery|Yes|Yes|
|Qos Strategy|TCP/UDP|Rich QoS Policies|
|AUTOSAR|CP-support/AP|CP-most/AP-support|
|Clouding|Nonsupport|Need DDS Web transfer|
|Security|TLS|DDS Security Standard, Supports fine-grained security rules and also supports TLS |
|Application Area|Automobile|Industrial, aviation, automotive, etc|
