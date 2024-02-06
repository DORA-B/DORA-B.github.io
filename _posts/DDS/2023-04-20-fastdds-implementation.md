# The DCPS conceptual model

DDS is a Data-Centric Publish Subscribe(DCPS) model, and three key application entities are defined in its iimplementation: 
1. publiucation entities
2. Sbuscription entities
3. configuration entities

+ Publisher: It implements DataWriter. Each one will have an assigned Topic UNDER which the messages are published.
+ Subscriber: DCPS Entity in charge of receiving the data published under the topics to which it subscribes. Serves one or more DataReader objects --> which communicating the avaliablity of new data to the application.
+ Topic: The Entity that binds publications and subscriptions. Being unique in a DDS domain. Through the TopicDescription, it allows the uniformity of data types of publications and subscriptions.
+ Domain: A concept used to link all publishers and subscribers, belonging to one or more applications, which exchange data under different topics. These individual applications that participate in a domain called DomainParticipant.
The DDS Domain is identified by a domain ID, DomainParticipant defines the domain ID to specify the DDS domain to which belongs. And with different IDs(DDS Domain IDs), two DomainParticipant are not aware of each other's  presence in the network --> In that case, if they want to communicate with each other, some connecting channels may be created, however, during communication, these applications must not interfere.
The DomainParticipant acts as a container for other DCPS Entities, acts as a factory for Publisher, Subscriber and Topic Entities, and provides administrative services in the domain. 



![DCPS model entities in the DDS Domain](/commons/images/3141911-20230414183838336-1640883136.png)


# Architecture
The architecture of Fast DDS is a layer model.
+ Application layer: user application that makes use of the Fast DDS API for the implementation of communications in distributed systems.
+ Fast DDS layer. Implementation of the DDS communications middleware. It allows the deployment of one or more DDS domains in which DomainParticipants within the same domain exchange messages by publishing/subscribing under a domain topic.
+ RTPS layer. Implementation of the Real-Time Publish-Subscribe (RTPS) protocol for interoperability with DDS applications. This layer acts an abstraction layer of the transport layer.
+ Transport Layer. Fast DDS can be used over various transport protocols such as unreliable transport protocols (UDP), reliable transport protocols (TCP), or shared memory transport protocols (SHM).

![Fast DDS layer model architecture](/commons/images/3141911-20230418114713749-672995737.png)
# DDS layer
The user will create these elements in their application, thus inmcorporating DDS application elements and creating a data-centric communication as Entities. 

A DDS Entity is any object that support Quality of Service configuration(Qos), and that implements as a listener.
(QoS: The mechanism by which the behavior of each of the entities is defined. Listener: The mechanism by which the entities are notified of the possible events that arise during the application’s execution.)

DDS Entities together with their their description and functionality.

**Below are described in a `DDS Layer API model` way**

+ Core(API module). It defines the abstract classes and interfaces that are refined by the other modules. It also provides the Qos definitions, as well as support for the notification-based interaction style with the middleware.

+ Domain. A positive integer which identifies the DDS domain. Each DomainParticipant will have an assigned DDS domain, so that DomainParticipants in the same domain can communicate, as well as isolate communications between DDS domains. This value must be given by the application developer when creating the DomainParticipants.
    + DomainParticipant. Object containing other DDS entities such as publishers, subscribers, topics and multitopics. It is the entity that allows the creation of the previous entities it contains, as well as the configuration of their behavior.DomainParticipant acts as an entry-point of the Service, as well as factory for many classes. The DomainParticipant also acts as a container for the other objects that make up the Service.

+ Publisher. In the implement of the DDS layer API, Publisher contains Publisher and DataWriter classes and 2 interfaces:
    + Publisher. The Publisher publishes data under a topic using a DataWriter, which writes the data to the transport. It is the entity that creates and configures the DataWriter entities it contains, and may contain one or more of them.
    + DataWriter. It is the entity in charge of publishing messages. The user must provide a Topic when creating this entity which will be the Topic under which the data will be published. Publication is done by writing the data-objects as a change in the DataWriterHistory.
    + PublisherListener(Interface, Not Entity)
    + DataWriterListener(Interface, Not Entity)
    
+ DataWriterHistory(Not API mode). This is a list of changes to the data-objects. When the DataWriter proceeds to publish data under a specific Topic, it actually creates a change in this data. It is this change that is registered in the History. These changes are then sent to the DataReader that subscribes to that specific topic.

+ Subscriber. Describes the classes used on the subscroption side, including two classes and two interfaces:
    + Subscriber. The Subscriber subscribes to a topic using a DataReader, which reads the data from the transport. It is the entity that creates and configures the DataReader entities it contains, and may contain one or more DataReader entities.
    + DataReader. It is the entity that subscribes to the topics for the reception of publications. The user must provide a subscription Topic when creating this entity. A DataReader receives the messages as changes in its HistoryDataReader.
    + SubscriberListener(Interface, Not Entity)
    + DataReaderListener(Interface, Not Entity)

+ DataReaderHistory(Not API mode). It contains the changes in the data-objects that the DataReader receives as a result of subscribing to a certain Topic.

+ Topic. Describes the classes used to define communication topics and data types. Including two classes and two interfaces:
    + Topic. Entity that binds Publishers’ DataWriters with Subscribers’ DataReaders.  
    + TopicDescription. class. 
    + TypeSupport and TopicListener (Interfaces)

# RTPS layer
The lower RTPS layer of eprosima Fast DDS serves an implementation of the protocol defined in the RTPS standard. This layer provides more control over the internals of the communication protocol than the DDS layer.

## Relation to the DDS layer
Elements of this layer map one-to-one with elements from the DDS layer, with a few 



# Transport layer

Fast DDS supports the implementation of applications over various transport protocols. -->
UDPv4;UDpv6; TCPv4; TCPv6; Shared Memory Transport(SHM).

By default, a DomainParticipant imiplements a UDPv4 and a SHM transport protocol. 

# RTPS 

+ Developed to support DDS applications, is a publication-subscription communication middleware over best-effort transports such as UDP/IP. 
+ Fast DDS provides support for TCP and Shared Memory(SHM) transports.
+ It is designed to support both unicast and multicast communications. 
+ At the top of RTPS, the Domain can be found.(Inherited from DDS, which defines a separate plane of commuinication)
+ Several Domains can coexist at the same time independently. A domain contains any number of RTPSParticipants, that is, elements capable of sending and receiving data. To do this, the RTPSparticipants use their Endpoints.

  + RTPSWriter: Endpoint able to send data.
  + RTPSReader: Endpoint able to receive data.

In the RTPS Domain also contain RTPSDomain and Participant put forward before.



![RTPS high-level architecture](/commons/images/3141911-20230417102515277-1096147059.png)

The topics do not belong to a specific participant. The participant, through the RTPSWriters, makes changes in the data published under a topic, and through the RTPSReaders receives the data associated with the topics to which it subscribes. 

The topics do not belong to a specific participant. The participant, through the RTPSWriters, makes changes in the data published under a topic, and through the RTPSReaders receives the data associated with the topics to which it subscribes. 

RTPSReaders/RTPSWriters register these changes on their History, a data structure that serves as a cache for recent changes.
## RTPS layer

The RTPS protocol in Fast DDS allows the absraction of DDS application entities from the transport laye. According to the graph shown above, the RTPS layer has four main Entities.

+ RTPSDomain. It is the extension of the DDS domain to the RTPS protocol.

+ RTPSParticipant. Entity containing other RTPS entities. It allows the configuration and creation of the entities it contains.

+ RTPSWriter. The source of the messages. It reads the changes written in the DataWriterHistory and transmits them to all the RTPSReaders to which it has previously matched.

+ RTPSReader. Receiving entity of the messages. It writes the changes reported by the RTPSWriter into the DataReaderHistory.

# The default configuration of eProsima Fast DDS

In the default configuration of eProsima Fast DDS, when you publish a change through a RTPSWriter endpoint, the following steps happen behind the scenes:

1. The change is added to the RTPSWriter¡¯s history cache.
2. The RTPSWriter sends the change to any RTPSReaders it knows about.
3. After receiving data, RTPSReaders update their history cache with the new change.

Still, there are numerous configurations that allow differe behaviors of RTPSWriters/RTPSReaders, which change the data exchange flow between RTPSWriters and RTPSReaders.


# Programming and execution model

Fast DDS is concurrent and event-based. This is also a multithreading model.

## Concurrency and multithreading

Fast DDS implements a concurrent multithreading system. 
Each DomainParticipant spawns a set of threads to take care of background tasks such as logging, message reception, and asynchronous communication. 
The Fast DDS API is thread safe, so just call any methods on the same DomainParticipant from different threads.
However, this multithreading implementation must be taken into account when external functions access to resources that are modified by threads running internally in the library(such as modified resources in the entity listener callbacks).

Brief overview of how Fast DDS multithreading schedule work.

+ Main thread: Managed by the application.

+ Event thread: Each DomainParticipant owns one of these. It processes periodic and triggered time events.

+ Asynchronous writer thread: This thread manages asynchronous writes for all DomainParticipants. Even for synchronous writers, some forms of communication must be initiated in the background.

+ Reception threads: DomainParticipants spawn a thread for each reception channel, where the concept of a channel depends on the transport layer (e.g. a UDP port).

## Event-driven architecture

There is a time-event system that enables Fast DDS to respond to certain conditions, as well as schedule periodic oprations.

Few of them are visible to the user since most are related to DDS and RTPS metadata. However, the user can define in their application periodic time-events by inheriting from the TimedEvent class.

# Functionalities

Fast DDS has some added features that can be implemented and configured by the user in the their application.

## Discovery Protocols
The discovery protocols define the mechanisms by which DataWriters publishing under a given Topic, and DataReaders subscribing to that same Topic are matched, so that they can start sharing data. 

+ Simple Discovery. default discovery mechanism, which is defined in the RTPS standard and provides compatibility with other DDS implementations. Here the DomainParticipants are discovered individually at an early stage to subsequently match the DataWriter and DataReader they implement.

+ Discovery Server. This discovery mechanism uses a centralized discovery architecture, where servers act as hubs for meta traffic discovery.

+ Static Discovery. This implements the discovery of DomainParticipants to each other but it is possible to skip the discovery of the entities contained in each DomainParticipant (DataReader/DataWriter) if these entities are known in advance by the remote DomainParticipants.

+ Manual Discovery. This mechanism is only compatible with the RTPS layer. It allows the user to manually match and unmatch RTPSParticipants, RTPSWriters, and RTPSReaders using whatever external meta-information channel of its choice.

## About Logging
Log class is the entry point fo the Logging system. It exposes three macro definitions to ease its usage: `EPROSIMA_LOG_INFO, EPROSIMA_LOG_WARNING and EPROSIMA_LOG_ERROR`.
Allows the definitions of new categories. 
Provides filtering by category using regular expressions.
control of the verbosity of the logging system.

## XML profiles configuration
With the existing of XML profile configuration, Fast DDS's default settings can be changed.
--> behaviors of the DDS Entities can be modified without the need for user to implement any program source code of re-build existing application.

## Environment variables

Env Vars: define outside the scope of the program, through operating system functionalities. 


# A simple C++ publisher and subscriber application

The code is here: [https://fast-dds.docs.eprosima.com/en/latest/fastdds/getting_started/simple_app/simple_app.html]

```cmd
.
└── workspace_DDSHelloWorld
    ├── build
    │   ├── CMakeCache.txt
    │   ├── CMakeFiles
    │   ├── cmake_install.cmake
    │   ├── DDSHelloWorldPublisher
    │   ├── DDSHelloWorldSubscriber
    │   └── Makefile
    ├── CMakeLists.txt
    └── src
        ├── HelloWorld.cxx
        ├── HelloWorld.h
        ├── HelloWorld.idl
        ├── HelloWorldPublisher.cpp --> Publisher
        ├── HelloWorldPubSubTypes.cxx
        ├── HelloWorldPubSubTypes.h
        └── HelloWorldSubscriber.cpp --> Subscriber
```


+ Using CMakeLists.txt to construct the whole project.
+ Fast DDS Publisher and Subscriber are under the same directory, and their code construction are almost the same.
+ In the HelloworldPublisher's init function, seven things are done
```cpp
1. Initializes the content of the HelloWorld type hello_ structure members.

2. Assigns a name to the participant through the QoS of the DomainParticipant.

3. Uses the DomainParticipantFactory to create the participant.

4. Registers the data type defined in the IDL.

5. Creates the topic for the publications.

6. Creates the publisher.

7. Creates the DataWriter with the listener previously created.
```
+ using CMakeList.txt to build them executable, and links the excutable and the library together.

```cmd
quintin@ubuntu:~/Documents/FstDds/DDSHello/src$ ls
CMakeCache.txt  cmake_install.cmake     DDSHelloWorldSubscriber  HelloWorld.h    HelloWorldPublisher.cpp    HelloWorldPubSubTypes.h   Makefile
CMakeFiles      DDSHelloWorldPublisher  HelloWorld.cxx           HelloWorld.idl  HelloWorldPubSubTypes.cxx  HelloWorldSubscriber.cpp
quintin@ubuntu:~/Documents/FstDds/DDSHello/src$ ./DDSHelloWorldPublisher 
Starting publisher.
Publisher matched.
Message: HelloWorld with index: 1 SENT
Message: HelloWorld with index: 2 SENT
Message: HelloWorld with index: 3 SENT
Message: HelloWorld with index: 4 SENT
Message: HelloWorld with index: 5 SENT
Message: HelloWorld with index: 6 SENT
Message: HelloWorld with index: 7 SENT
Message: HelloWorld with index: 8 SENT
Message: HelloWorld with index: 9 SENT
Message: HelloWorld with index: 10 SENT
Publisher unmatched.
```

```cmd
quintin@ubuntu:~/Documents/FstDds/DDSHello/src$ cmake --build .
Consolidate compiler generated dependencies of target DDSHelloWorldPublisher
[ 50%] Built target DDSHelloWorldPublisher
[ 62%] Building CXX object CMakeFiles/DDSHelloWorldSubscriber.dir/HelloWorldSubscriber.cpp.o
[ 75%] Building CXX object CMakeFiles/DDSHelloWorldSubscriber.dir/HelloWorld.cxx.o
[ 87%] Building CXX object CMakeFiles/DDSHelloWorldSubscriber.dir/HelloWorldPubSubTypes.cxx.o
[100%] Linking CXX executable DDSHelloWorldSubscriber
[100%] Built target DDSHelloWorldSubscriber
quintin@ubuntu:~/Documents/FstDds/DDSHello/src$ ./DDSHelloWorldSubscriber 
Starting subscriber.
Subscriber matched.
Message: HelloWorld with index: 1 RECEIVED.
Message: HelloWorld with index: 2 RECEIVED.
Message: HelloWorld with index: 3 RECEIVED.
Message: HelloWorld with index: 4 RECEIVED.
Message: HelloWorld with index: 5 RECEIVED.
Message: HelloWorld with index: 6 RECEIVED.
Message: HelloWorld with index: 7 RECEIVED.
Message: HelloWorld with index: 8 RECEIVED.
Message: HelloWorld with index: 9 RECEIVED.
Message: HelloWorld with index: 10 RECEIVED.
```

# A simple Python publisher and subscriber application

structure is like below:
```cpp
.
├── CMakeCache.txt
├── CMakeFiles
├── CMakeLists.txt
├── HelloWorld.cxx
├── HelloWorld.h
├── HelloWorld.i
├── HelloWorld.idl
├── HelloWorld.py
├── HelloWorldPubSubTypes.cxx
├── HelloWorldPubSubTypes.h
├── HelloWorldPubSubTypes.i
├── HelloWorldPublisher.py
├── HelloWorldSubscriber.py
├── Makefile
├── _HelloWorldWrapper.so
├── cmake_install.cmake
└── libHelloWorld.so
```

**What is special?**
using SWIG interface files for the Python bindings. 
> fastddsgen -python HelloWorld.idl

File [HelloWorld.i]: SWIG interface file for HelloWorld C++ type definition.

Remember to install Fast-DDS-Python first, following the instruction in [https://fast-dds.docs.eprosima.com/en/latest/installation/sources/sources_linux.html]

```cpp
// Copyright 2016 Proyectos y Sistemas de Mantenimiento SL (eProsima).
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

/*!
 * @file HelloWorld.i
 * This header file contains the SWIG interface of the described types in the IDL file.
 *
 * This file was generated by the tool gen.
 */

%module HelloWorld

// SWIG helper modules
%include "stdint.i"
%include "std_string.i"
%include "std_vector.i"
%include "std_array.i"
%include "std_map.i"
%include "typemaps.i"

// Assignemt operators are ignored, as there is no such thing in Python.
// Trying to export them issues a warning
%ignore *::operator=;

// Macro declarations
// Any macro used on the Fast DDS header files will give an error if it is not redefined here
#define RTPS_DllAPI
#define eProsima_user_DllExport


%{
#include "HelloWorld.h"

#include <fastdds/dds/core/LoanableSequence.hpp>
%}

%import(module="fastdds") "fastdds/dds/core/LoanableCollection.hpp"
%import(module="fastdds") "fastdds/dds/core/LoanableTypedCollection.hpp"
%import(module="fastdds") "fastdds/dds/core/LoanableSequence.hpp"

////////////////////////////////////////////////////////
// Binding for class HelloWorld
////////////////////////////////////////////////////////

// Ignore overloaded methods that have no application on Python
// Otherwise they will issue a warning
%ignore HelloWorld::HelloWorld(HelloWorld&&);

// Overloaded getter methods shadow each other and are equivalent in python
// Avoid a warning ignoring all but one
%ignore HelloWorld::index(uint32_t&&);

// Overloaded getter methods shadow each other and are equivalent in python
// Const accesors produced constant enums instead of arrays/dictionaries when used
// We ignore them to prevent this
%ignore HelloWorld::index();
%rename("%s") HelloWorld::index() const;

%ignore HelloWorld::message(std::string&&);

// Overloaded getter methods shadow each other and are equivalent in python
// Const accesors produced constant enums instead of arrays/dictionaries when used
// We ignore them to prevent this
%ignore HelloWorld::message();
%rename("%s") HelloWorld::message() const;


%template(_HelloWorldSeq) eprosima::fastdds::dds::LoanableTypedCollection<HelloWorld, std::false_type>;
%template(HelloWorldSeq) eprosima::fastdds::dds::LoanableSequence<HelloWorld, std::false_type>;
%extend eprosima::fastdds::dds::LoanableSequence<HelloWorld, std::false_type>
{
    size_t __len__() const
    {
        return self->length();
    }

    const HelloWorld& __getitem__(size_t i) const
    {
        return (*self)[i];
    }
}
// Include the class interfaces
%include "HelloWorld.h"

// Include the corresponding TopicDataType
%include "HelloWorldPubSubTypes.i"

```
