# Data Distribution Service
To solve the problem when massive data is acquired to be distributed Real-time, efficiently, flexibly. 
Data is the center.

Adaptive AUTOSAR is the first company that applied DDS as one of the optional communication methods.
ROS2 also used DDS to deliever the data.
eProsima's Fast DDS is the standard c++ implement in FastDDS.

![Where is DDS in the Adaptive Application](/commons/images/3141911-20230413094244547-295714901.png)

# DDS standard
OMG（Object Management Group）

DDS's Relevant standards include core protocols(DDSI-RTPS，DDS-XTypes，DDS-Security，Interface Definition Language (IDL)…)，API(DDS C++ API，DDS Java API),Extension agreement(DDS-RPC,DDS-xml).

# DDS in the distributed system
In the distributed system, DDS is between the system and the application, which Supports multiple programming languages and multiple underlying protocols.
![Where is DDS in the distributed system](/commons/images/3141911-20230413100406198-2038202766.png)

# DDS communicational model DCPS


![DCPS model entities in the DDS Domain.](/commons/images/3141911-20230414183532916-1433264985.png)
Domain Participant: Represents the local member identity of the application program communicating within the domain. In simple terms, it explains the communication members within the same data domain.

Topic: Is the abstract concept of data, identified by TopicName, associated with the corresponding data type (DataType). If all the Topics involved in a vehicle are collected together, it forms a virtual global data space called "Global Data Space", further weakening the concept of nodes. Therefore, Domain Participants are no longer the concept of nodes.

DataWriter: The data writer, similar to a cache, writes the topic data that needs to be published from the application layer into the DataWriter.

DataReader: The data reader, also can be understood as a type of cache, obtains the topic data from the subscriber and passes it to the application layer.

Publisher: The publisher publishes the topic data, associated with at least one DataWriter, and sends the data by calling the relevant functions of the DataWriter.

Subscriber: The subscriber subscribes to the topic data, associated with at least one DataReader. When the data arrives, the application program may be busy executing other operations or simply waiting for the message, which leads to two situations: synchronous access and asynchronous notification.

User only care about the data they use, in that case, DCPS(Data-Centric Pubilish-Subscribe), package the data to a service, afterwards, the consumer direction of service is subscribed by the service provider through SD to the event group in the service. When the data changes, the corresponding event message will be sent to the bus.

![DDS model](/commons/images/3141911-20230413100745917-1326414217.png)
![DDS description from article](/commons/images/3141911-20230414145641355-682665420.png)

# QoS(Quality of Service)

An important knowedge about the DDS is that it can support QoS(Quality of Service). Currently, there are 22 QoS policies supported, and each policy can be applied to different roles. For the same role, a single QoS policy can be used alone or multiple QoS policies can be combined.

![22 Qos policies](/commons/images/3141911-20230413104225258-727070958.png)

```cmd
RELIABILITY

Parameter definitions:

Kind = RELIABLE, if DataReader may not be able to receive the sample data of DataWriter when an error occurs in the network, the sample data will be resent to ensure that DataReader can receive the data;

Kind = BEST_EFFORT, if a network error occurs, DataWriter will not resend the missing sample data, so there is no guarantee that DataReader will receive the data;

If this QoS policy is applied on DataWriter, set Kind = RELIABLE, which ensures that the data published by DataWriter can be received by DataReader.
```
```cmd
LIFESPAN (Life Cycle)

Parameter definitions:

The function of this QoS is to avoid the delivery of "expired" data, the parameter is time duration, the default is infinity, indicating that the data sample will never expire; If duration is set to a finite value, and the clocks of the sender and receiver are synchronized at the same time, by adding a defined duration field to the source timestamp of the sender, the receiver calculates whether the data is invalid based on the timestamp information, and if it fails, the data can be deleted directly.
```
***
Blow are the description of the implementation of the DDS --- eProsima
***

# DDS API
 
1. Many to Many
2. Regulated by Qos
3. data-centric model
4. Publishers -> data space -> subscribers, and each time middleware propagates the information to all interested subscribers.
5. Comminications happens across domains, Only entities belonging to a same domain can interact.
6. Topics are unambiguous identifiers(unique in a domain), which mediates matching between entities subscirbing to data and entities publishing them.
7. DDS entities are modeled either as classes(case.1) or typed interfaces(case.2), and case.2 imply a more efficient resource handling. Because as knowledge of the data type prior to the execution allows allocating memory in advance rather than dynamically.

![2 different model of DDS entities](/commons/images/3141911-20230413111148944-1208744745.png)

## A Design of high level API
The basic class hierarchy is depicted in Figure6. An abstract class named“DDS_Interface” that is inherited by two subclasses named “DDS_Entity_Pub” and “DDS_Entity_Sub”
### DDS hierarchy diagram
![Basic class hierarchy diagram](/commons/images/3141911-20230414150107648-9436316.png)

### DDS entity publication
![DDS interface](/commons/images/3141911-20230414150148211-473674334.png)
### DDS enetity Subscirber
![interfaces of the Subscriber Entity](/commons/images/3141911-20230414150347059-652180754.png)
# Fast DDS-Gen

1. A need for a generation tool that translates type descriptions into appropriate implementations that fills the gap between the interfaces and the middleware.
2. Fast DDS-Gen, a implemented version which is a Java application that generates source code using the data types defined in an Interface Definition Language(IDL) file.

```cpp
// cmd command
//> mkdir -p dds_helloword/src/helloworld && cd dds_helloword/src/helloworld
//> gedit HelloWorld.idl
/* HelloWorld definition */
struct HelloWorld
{
    unsigned long index;
    string message;
};

// generate .h and .cxx files.
// > fastddsgen HelloWorld.idl
```
there is another way to generate code

```cmd
// using '-example' option to enable a specifical platform 
// fastddsgen -example Cmake Helloworld.idl
├── CMakeLists.txt
├── HelloWorld.cxx
├── HelloWorld.h
├── HelloWorldPublisher.cxx --> publisher
├── HelloWorldPublisher.h
├── HelloWorldPubSubMain.cxx --> main
├── HelloWorldPubSubTypes.cxx
├── HelloWorldPubSubTypes.h
├── HelloWorldSubscriber.cxx --> Subscriber
└── HelloWorldSubscriber.h
```

If Logs are required, refer to `test/unittest/logging/LogTests.cpp`
Info logs will reveal only if the switch is on: [Debug mode on]: LOG_NO_INFO

# RTPS Wire Protocol

1. Real-Time Publish-Subscribe protocol (RTPS) which provides publisher-subscriber communications over transports such as TCP/UDP/IP, and guarantees compatibility among different DDS implementations.

2. RTPS's designed for meeting the same requirements addressed by the DDS application domain. This protocol maps to many DDS concepts, is a natural implementation for DDS.

3.  All the RTPS core entities are associated with an RTPS domain, which represents an isolated communication plane where endpoints match.

4. The entities specified in the RTPS protocol are in one-to-one correspondence with the DDS entities, thus allowing the communication to occur.


## What are the differences between DDS and RTPS?

The entities specified in the RTPS protocol are in one-to-one correspondence with the DDS entities, thus allowing the communication to occur.

RTPS stands for Real-Time Publish Subscribe. It is now more formally known as OMG DDSI-RTPS, where DDSI stands for DDS Interoperability. It is standardized by the Object Management Group (OMG) with the purpose to provide an interoperability wire protocol for DDS implementations.

DDS stands for Data Distribution Service. It is a data-centric connectivity framework based on the publish-subscribe paradigm. The OMG DDS specification describes an API as well as the expected behavior of DDS infrastructures. It provides its users with advanced data-management features, including a comprehensive type system, multiple communication patterns and different types of Quality of Service, which simplify the task of building distributed (real-time) systems.

It is common, although not required, for DDS implementations to leverage the DDSI-RTPS wire protocol under the hood to achieve the required inter-process (network) communications. DDS users do not have to be aware of the inner workings of RTPS, although it is possible get a glimpse by using Wireshark, which comes with an RTPS dissector. If the requirements include combining applications built with different DDS implementations in a single system, then RTPS support is a must.

It is also possible to use RTPS as a protocol by itself, without a full DDS implementation on top of it and without the use of a standardized API. That does not provide many of the DDS-features that I mentioned earlier, but only the lower level publish-subscribe functionality.

Here is a more specified intro.
[https://stackoverflow.com/questions/59278753/difference-between-dds-and-rtps]

[https://wiki.wireshark.org/Protocols/rtps]

# Main features

Actually, there are lots of excellent features, blow are some important.

+ Two API Layers. eProsima Fast DDS comprises a high-level DDS compliant layer focused on usability and a lower-level RTPS compliant layer that provides finer access to the RTPS protocol.
+ Real-Time behaviors. Guaranteeing responses within specified time constrains.
+ Best effort and reliable communication.
+ Sync and Async publication modes.
+ Security.
+ Flow controllers.
+ Scalability and Flexibility. DDS builds on the concept of a global data space. The middleware is in charge of propagating the information between publishers and subscribers. This guaratees that the distributed network is adaptable to reconfigurations and scalable to a large number of entities.
