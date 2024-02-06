---
title: Project Overview
date: 2023-10-02
categories: [AutoSAR]
tags: [autosar, project]     # TAG names should always be lowercase
published: false
---

# Contents

[TOC]

# Overview of the whole project

![Technical Architecture of Communication Management](/commons/images/3141911-20230504144449396-1988547944.png)

After compiling, the outputs are in directory ./ara-api/build and ./output/

```txt 
.
├── apd
│   ├── etc
│   ├── include
│   ├── lib
│   ├── opt
│   ├── sbin
│   └── share
├── AP_Project
│   ├── event_method_field
│   ├── only_event
│   ├── per_test
│   └── two_machine
├── ara-api
│   ├── apd
│   ├── build                    <--constructed files-->
│   ├── CMakeLists.txt
│   ├── com                      <--focus content: communication-->
│   ├── core
│   ├── diag
│   ├── exec
│   ├── iam
│   ├── log
│   ├── per
│   ├── phm
│   ├── scheduler
│   ├── sm
│   ├── tools
│   ├── tsync
│   ├── ucm
│   └── w3c_license.txt
├── dds
│   ├── build.sh
│   ├── memory
│   ├── Power-CDR
│   ├── Power-DDS
│   ├── Power-DDS-Gen
│   ├── ReadMe
│   └── Tool.py
├── output                      <--constructed files-->
│   ├── bin
│   ├── etc
│   ├── include
│   ├── lib
│   ├── opt
│   ├── sbin
│   └── share
├── projecttree.txt
├── sample-applications
│   ├── applications
│   ├── CMakeLists.txt
│   ├── copy_idl_genfiles.sh
│   ├── diag_samples
│   ├── integration_scenarios
│   ├── interfaces
│   ├── machine_manifest
│   ├── machines
│   ├── network
│   ├── per_examples
│   ├── README.md
│   ├── safety_app
│   ├── sec_examples
│   ├── st_scenarios
│   ├── system_deinit
│   ├── system_init
│   └── wrsomeip_examples
└── utils
    ├── 3rd-party
    ├── apgen
    ├── applications
    ├── genRuntime.py
    ├── include
    ├── lib
    ├── LinearxGen
    ├── LineaxBuild
    ├── Readme
    ├── script
    └── share

64 directories, 13 files
```
# Acronyms and Abbreviations of Communication Management

|Abbreviation/Acronym:|Description:|
|:---|:---|
|CM| Communication Management|
|IP| Internet Protocol|
|SOME/IP| Scalable service-Oriented |MiddlewarE over IP|
|TCP| Transmission Control Protocol|
|UDP| User Datagram Protocol|
|E2E| End-to-end communication protection|
|SoC| Service-Oriented Communication|
|SecOC| Secure Onboard Communication|
|DTLS| Datagram Transport Layer Security|
|DDS| Data Distribution Service|
|RTPS| Real Time Publish Subscribe Protocol|
|TTL| Time To Live|
|TLV| Tag-Length-Value
|RPC| Remote Procedure Call|
|QoS| Quality of Service|
|BOM| Byte Order Mark|

|Term:| Description:|
|:---|:---|
|Callable |In the context of C++ a Callable is defined as: A Callable type is a type for which the INVOKE operation (used by, e.g., std::function, std::bind, and std::thread::thread) is applicable. This operation may be performed explicitly using the library function std::invoke. (since C++17)|
|serializedSample |A serializedSample is the serialization of a C++ object to an array and consists of the header that is part of e2e protection and the serialized data.|
|Service Binding| Act of connecting a Service Requester to a concrete Service Instance of a Service Provider.|
|Multi-Binding| Multi-Binding describes setups having multiple connections implemented by different technical transport layers and protocol between different instances of a single proxy or skeleton class, e.g.: ==• A proxy class uses different transport/IPC to communicate with different skeleton instances. • Different proxy instances for the same skeleton instance uses different transport/IPC to communicate with this instance: The skeleton instance supports multiple transport mechanisms to get contacted==|
# Functional Specification
The AUTOSAR Adaptive architecture organizes the software of the AUTOSAR Adaptive foundation as functional clusters. These clusters offer common functionality as services to the applications. 
The Communication Management (CM) for AUTOSAR Adaptive is such a functional cluster and is part of "AUTOSAR Runtime for Adaptive Applications" - ARA. It is responsible for the construction and supervision of communication paths between applications, both local and remote.
The CM provides the infrastructure that enables communication between Adaptive AUTOSAR Applications within one machine and with software entities on other machines, e.g. other Adaptive AUTOSAR applications or Classic AUTOSAR SWCs. All communication paths can be established at design- , start-up- or run-time
This specification includes the syntax of the API, the relationship of API to the model and describes semantics, e.g. through state machines, and assumption of pre-, postconditions and use of APIs. The specification does not provide constraints on the SW architecture of a platform implementation, so there is no definition of basic software modules and no specification of implementation or internal technical architecture of the Communication Management.

## Architectural concepts

COM (COMmunication) - Sitting between the RTE and the PDU Router, it defines all the signals used by the port in the Software component, and converts them into from data structures into a PDU, and vice-versa. Aside from this, it can group PDU's and allow you to control these groups, activating or deactivating them. It does not care about the underlying protocol the message is being transmitted from or received to.

The Communication management of AUTOSAR Adaptive can be logically divided into
the following sub-parts:

+ Language binding
+ End-to-end communication protection
+ Communication / Network binding
+ Communication Management software

![Technical Architecture of Communication Management](/commons/images/3141911-20230505140448421-200624965.png)
In the context of Communication Management, the following types of interfaces are defined:
+ Public Application Interface: Part of the Adaptive AUTOSAR API and specified in the SWS. This is the standardized ara::com API.
+ Functional Cluster Interactions: Interaction between functional clusters. Not normative, intended to make specification more readable and to support integration of SW into demonstrator. (dotted arrow in figure) And also interactions between elements within a functional cluster. Not used in specifications, so it is a nonstandardized interface. Used for communication inside Communication Management software (grey arrow in figure).

Language Binding and Communication Binding depend on a specific configuration by the integrator, but they need to be deployed within the application binary. This results in the fact that the serialization of the Communication Binding will run in the execution context of the Adaptive Application.

==<font face='TIMES' size='5'> Following constraints are applied to the design of ARA API </font>==
+ Support the independence of application software components
+ Use of Service-oriented communication without dependency on a specific communication protocol
+ Make the API as lean as possible, neither supporting very specific use cases which could also be done on the top of the API, nor supporting components model or highter level concepts. The API is restricted to support core communication mechanisms.
+ Support for dynamic communication:
    - No discovery by application middleware, the clients know the sever but the Server does not know the clinets. Event subscription is the only dynamic communication pattern in the application.
    - Full service discovery in the application. No communication paths are known at configuration pattern in the application.
+ Support both Event/Callback and Polling style usage of the API to enable calssic RTE style paradigms. To support high determinism demands in case of callback-based/event-based interaction, there shall be the possibility to avoid uncontrolled context switches.

+ Support both synchronous callback-based communication and asynchronous communication philosophy.

+ Support of client/sever communication.

+ Support of selection of trigger conditions for task activation.

+ Extensions for security.

+ Extensions for Quality Of Service QoS.

+ Scalability of built-in end-to-end communication protection, where a use-case-specific behavior can be done on top of ARA API.
## Design decisions
The design of the ARA API convers the following principles:
+ It uses the Proxy/Skeleton pattern:
    - The (service) proxy is the representative of the possibly remote (i.e. other process, other core other node).
    - The (service) skeleton is the connection of the user provided service implementation to the middleware transport infrastructure. Service implementation class is derived from the (service) skeleton.
    - Beside proxies/skeletons, there might exist a so-called "Runtime"(singleton) class to provide some essentials to manage proxies and skeletons. But this is communication management software imlementation specific and therefore not specified here, but may be specified in a future version.

    Middleware is reusable software that leverages patterns and frameworks to bridge the gap between the functional requirements of applications and the underlying operating systems, network protocol stacks, and databases. This paper presents an overview of patterns, frameworks, and middleware, describes how these technologies complement each other to enhance reuse and productivity, and then illustrates how they have been applied successfully in practice to improve the reusability and quality of complex software systems.

    regarding proxy/skeleton design pattern in general and its role in middleware implementations: 

    paper1: [Middleware for real-time and embedded systems](https://dl.acm.org/doi/10.1145/508448.508472),
    ![paper1](/commons/images/3141911-20230505153905396-362816222.png)
    paper2: [Patterns, frameworks, and middleware: their synergistic relationships](https://dl.acm.org/doi/10.5555/776816.776917)
    ![paper2](/commons/images/3141911-20230505153954473-442285407.png)
+ It supports callback mechanisms on data reception.
+ The API has zero-copy capabilities including the possibility for memory management in the middleware.
+ It is aligned with the AUTOSAR service model(services, instances, events, methods, ...) to allow the generation of proxies and skeletons out of this model.
+ Full discovery and service instance selection support on API level.
+ Client/Sever Communication uses concepts introduced by C++11 language, e.g. std::future, std::promise, to fully support method calls between different contexts.
+ Abstract from SOME/IP specific behavior, bt support SOME/IP service mechanisms, as methods, events and fields.
+ Support/implement the standard end-to-end protection protocols, as specified in ==AUTOSAR_PRS_E2EProtocol[E2E Protocol Specification]== and 
==AUTOSAR_RS_E2E[Requirements on E2E]==
+ Support of Service contract versioning.
+ Support Event and Polling style usage of the API equally to enable classic RT style paradigms.
+ Fully exploit C++11/14 features in API design to provide usability and comfort for the application developer.
## Communication paradigms
Service-Oriented Communication(SoC) as a part of Service-Oriented Architecture (SOA) is the main communication pattern for Adaptive AUTOSAR Applications. It allows establishing communcation paths both at run-time, so it can be used to build up dynamic communication with unknown number of participants.
The basic operation principle of Service-Oriented Communication.
![image.png](/commons/images/3141911-20230505155724778-51599986.png)
Service Discovery decides whether external and internal service-oriented communication is established. The discovery strategy shall allow either returning a specific service instance or all available instances providing the requested service at the time of the request, no matter if they are available locally or remote. The Communication Management software should provide an optimized implementation for both the Service discovery and the communication connection, depending on the location where the service provider resides. More about Service Discovery can be found in SOME/IP Service Discovery Protocol Specification.

## Service contract versioning

In Service Oriented Architecture (SOA) environments the client and the provider of a service rely on a contract which covers the service interface and behavior. The interface and the behavior of a service may change over time. Therefore, service contract versioning has been introduced to differentiate between the different versions of a service.
![Service contract versioning over time](/commons/images/3141911-20230505160508602-110726420.png)

The AUTOSAR Adaptive platform supports service contract versioning. The service
contract versioning is separated between the design phase and the deployment phase.
This means that any service at design level may have its own version number which is mapped to a version number of the used network binding and vice versa. The mapping process is manually done by the service designer or integrator.

![Service contract versioning flow](/commons/images/3141911-20230505160704523-1195991657.png)

1. The contract version of a ==ServiceInterface== consist of a ==majorVersion== and a ==minorVersion== number. The ==majorVersion== number indicates backwards-incompatible service changes. The ==minorVersion== number indicates backwards-compatible service changes.
    + for backwards-incompatible interface or behavior changes the majorVersion number is increased and the ==minorVersion== number is set to 0.
    + for backwards-compatible interface or behavior changes the majorVersion number is unchanged and the ==minorVersion== number is increased.
2. The contract version of a ==ServiceInterface== is mapped to a version of the ==ServiceInterfaceDeployment==. This version mapping may be done several times resulting in several ==ServiceInterfaceDeployments== for the same ==ServiceInterface==. Such a mapping will result in unambiguous identification on each VLAN according to the [constr_1723] in ==AUTOSAR_TPS_ManifestSpecification[Specification of Manifest]==
# End-to-End communication protection for Events
(In the document AUTOSAR_SWS_CommunicationManagement)
## Limitations
The value of dataIdMode for Events and the notifier of Fields shall be set according to the dataIdMode of the E2EProfileConfiguration which is referenced (in role e2eProfileConfiguration) by the AdaptivePlatformServiceInstance.e2eEventProtectionProps which reference (in role event) the ServiceEventDeployment of the particular Event or the Field notifier.

## Publisher
The overview of the interaction of components involved during the E2E protection at the publisher side.
![E2E Publisher](/commons/images/3141911-20230505165651905-514237501.png)

## Subscriber - GetNewSamples
The overview of the interaction of components involved during the E2E
checking at the subscriber side.
![E2E Subscriber](/commons/images/3141911-20230505165932283-78630868.png)

# End-to-end communication protection for Methods
This section specifies the integration of E2E communication protection in ara::com for the processing of Methods. This includes E2E communication protection for a Method’s request as well as E2E communication protection for any kind of Method’s response (i.e., normal or error response).

## protection of the service method request (Client)
![Interaction of components during E2E protection of the Method request at the client side](/commons/images/3141911-20230506094025452-1275303321.png)

## E2E checking the service method request(Server)

==Interaction of components during E2E checking of the Method request at the server side - polling==
![Interaction of components during E2E checking of the Method request at the server side - polling](/commons/images/3141911-20230506094231520-1105777292.png)

==Interaction of components during E2E checking of the Method request at the server side - event driven==
![Interaction of components during E2E checking of the Method request at the
server side - event driven](/commons/images/3141911-20230506094449468-788937510.png)

## E2E protection of the service method response (Server)

For E2E-protected Methods, E2E protection of the response message shall be performed after the execution of the service method (in case of a successful E2E check according to [SWS_CM_90480]) or after the execution of the E2E error handler (in case of a failed E2E check according to [SWS_CM_90480]).
![Interaction of components during E2E protection of the Method response at the server side ](/commons/images/3141911-20230506094807115-1289194809.png)

## E2E checking the service method response (Client)

For E2E-protected Method responses, E2E checking shall be performed within the context of the message reception within the Service-Proxy(RS_CM_00400, RS_E2E_08541)

An overview of the interaction of components involved during the E2E checking of the Method response at the client side.
![Interaction of components during E2E checking of the Method response at the client side](/commons/images/3141911-20230506095107680-1437203095.png)

##  Timeout supervision

ara::com does not support any timeout supervision for method calls. A lost response message could block some ara::core::Future methods like wait() forever. In case of E2E such a timeout supervision is desired, wherefore the adaptive
application is strongly recommended to implement timeout supervision, e.g., by using the ReportCheckpoint() method of the ara::phm::SupervisedEntity or the wait_for(), wait_until(), or the is_ready() methods of the ara::core::Future.

# End-to-end communication protection for Fields
This section specifies E2E protection for fields. A field is a data object that can be accessed by a getter and/or setter method. In addition update notifications may be provided to subscribers, whenever the value of the field gets updated. The principle of fields is already specified. This section specifies the E2E protection for fields. The E2E protection for methods Get and Set follows the E2E protection for Methods.  The specifications [SWS_CM_10460] and [SWS_CM_90485] define the parameters for E2E protection of the methods Get() and Set(). 
The E2E protection for Update follows the E2E protection for events.The specifications [SWS_CM_90402] and [SWS_CM_90433] define the parameters for E2E protection of the update event. E2E results OK and OK_SOME_LOST are successful results. E2E results ERROR, REPEATED, WRONGSEQUENCE, NOTAVAILABLE and NONEWDATA are considered error results
There are E2E profiles 4m, 7m, 8m or 44m for the protection of methods (Get, Set). Also the other E2E profiles can be used for the protection of methods. But in this case some parameters of SOME/IP are not protected.

## Send a GET message

![Send a GET Message](/commons/images/3141911-20230506100016538-730933544.png)

## Receive a GET message
The message is received by the Publisher application. The Publisher application is a server application.
![ Receive a GET Message](/commons/images/3141911-20230506100131216-169571252.png)

## Receive a response to a GET message
If the message is received with an E2E error, then the E2E Errorhandler of the client is called. The future of the Get() function is set to ready state with an error code.
The received message is of type RESPONSE or ERROR (see [PRS_SOMEIP_00055]). Type ERROR indicates that an E2E error occurred at the server site. If a message of type ERROR is received with Return Code of E2E error (indicating that the Publisher received the Get request with an E2E error) then the E2E Errorhandler of the Client Application is called. The future of the Get() function is set to ready state with an error code.
If the RESPONSE message is received without E2E errors then the future is updated with the received value of the Publishers field. The future becomes ready and the Client application can use this value.

If a RESPONSE message to an outgoing Get message does not arrive at all, then the
client application is not informed if the value was retrieved from the remote application. The future of Field.Get() is not updated to state ready. In this case the client application can send the Get message again to the remote application to retrieve the value, or initiate its own error handling. A timeout supervision may unlock the future. 

Figure shows reception of a message from the server.
![Receive response to a GET Message](/commons/images/3141911-20230506100557830-416189190.png)

## Send a SET message

Figure shows the message flow of sending a Set() method. The figure does not list all details of E2E protection, e.g. functions of libraries E2ELib and CrcLib are omitted in this figure.
The client application calls the Set() function at ara::com with one argument (the value that shall overwrite the field’s value).
![Send a SET Message](/commons/images/3141911-20230506100721227-368453587.png)

## Receive a SET message

If the incoming message is received without E2E error the SetHandler of the Publisher application is called. The SetHandler returns the value to be written to the Publisher’s field. The returned value may be identical to the parameter of the SET message (successful update). But there is also the possibility that an update could not be performed completely. If the parameter of the SET message is out of range then the field may be left unchanged or the field is updated by a value inside the field’s range. The type of the response message is RESPONSE.
If the incoming message is received with an E2E error, then the Publisher is informed through the E2E error handler. In this case The SetHandler of the Publisher is not called. The type of the response message is ERROR. If the E2E_Check fails the Return Code of the ERROR message is initialized with an E2E error code (See [PRS_SOMEIP_00191])

The message to be returned (type ERROR or RESPONSE) is serialized, E2E protected and sent back to the client.

Through the figure. The E2E protected part of the serialized header is checked for E2E errors. If the incoming message was received with an E2E error, then the Publisher is informed through the E2E error handler. The Publisher’s field is not updated and no value is retrieved from Publisher’s field.
![ Receive a SET Message ](/commons/images/3141911-20230506101129208-548663585.png)
##  Receive a SET Message

Figure shows reception of a response. This message is of type RESPONSE or ERROR (see [PRS_SOMEIP_00055]) and similar to the reception of a response to a GET message.
![Receive response to a SET Message](/commons/images/3141911-20230506101312851-2074032817.png)

## Send an UPDATE message
An update of a subscriber is an event. The update message is sent to every subscriber to the publisher’s field.
Figure shows sending of field update messages.
![Send an UPDATE Message](/commons/images/3141911-20230506101413016-595581358.png)

## Receive an UPDATE message
If one or more update messages are received the E2E state machine provides one of the following results: OK, ERROR, REPEATED, NONEWDATA, WRONGSEQUENCE (See [PRS_E2E_00597]). Only result OK indicates that the received value is valid.

Figure shows reception of a field update message.
![Receive an UPDATE Message](/commons/images/3141911-20230506101623731-1034172183.png)

# Raw Data Streaming 
## Raw Data Streaming
## Raw Data Streaming
# Communication Group
## Interfaces
## Behavior
## Connection
## Limitations
## Communication Group Model
## Communication Group Creation

# Network Binding

## SOME/IP Network binding
##  Signal-Based Network binding
## DDS Network Binding


# ARA-COM contents (Focus Content)

![ara-com contents](/commons/images/3141911-20230425172740091-1429934546.png)

+ ./ARA/ara-api/com/include/public/ara/com/internal/proxy: A base class encapsulation for required services, mainly consists of event, method, field, proxy. Subsequent different communication protocols will inherit them to implement the actual subclasses event, method, field.

+ ./ARA/ara-api/com/include/public/ara/com/internal/skeleton: A base class encapsulation for provide services, mainly event, event_dispatcher, field, field_dispatcher, service_impl_base.

![The constructure of the dds_idl](/commons/images/3141911-20230504104438630-220583958.png)

+ ./ARA/ara-api/com/include/public/ara/com/internal/dds_idl: For the classes of the OpenDDS implementation, an encapsulation of OpenDDS, `<proxy>` and `<skeketon>` are also subclass concrete implementations.

==<font face='Times' size='5'>Communication design concept</font>==
The Skeleton side implements the service side of the corresponding app program, such as ==`<Radar>`==,

While the proxy side implements the client side of the corresponding app program, such as ==`<Fusion>`==.

## dds_idl in ara-com

[Where does the requirement come from?](https://www.autosar.org/fileadmin/standards/R21-11/AP/AUTOSAR_SWS_CommunicationManagement.pdf)

From the software specification file        `AUTOSAR_SWS_CommunicationManagement.pdf`.

![Abstract network protocol binding](/commons/images/3141911-20230504162414530-818602209.png)

Implement class diagram analysis for OpenDDS encapsulated classes and Event Method Field ServiceImpl in Skeleton.

![Implement class diagram analysis for OpenDDS](/commons/images/3141911-20230504110920224-664507236.png)


![Implement class diagram for DDS-skeleton](/commons/images/3141911-20230505113321470-335164588.png)


Radar and fusion sample:

/ARA/sample-applications/applications/fusion/src/main_fusion.cpp

/ARA/sample-applications/applications/radar/src/main_radar.cpp

Service registry:
/ARA/ara-api/com/src/libara_ddsidlbinding/service_registry.cpp

/ARA/ara-api/com/src/libara_fastddsbinding/fastdds_service_registry.cpp
