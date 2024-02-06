---
title: Introduction of AutoSAR
date: 2023-10-02
categories: [AutoSAR]
tags: [autosar, introduction]     # TAG names should always be lowercase
---
[toc]

# What is AutoSAR?

![img](/commons/images/3141911-20230424163854182-1819650861.png)

==[AUTOSAR](https://www.autosar.org/)== (AUTomotive Open System ARchitecture) is a worldwide development partnership of vehicle manufacturers, suppliers, service providers and companies from the automotive electronics, semiconductor and software industry.
Goals of AUTOSAR

+ to fullfil future vehicle requirements, such as availability and safety, SW upgrades/ updates, and maintainability
+ to increase scalability and flexibility to integrate and transfer functions
+ to increase a higher penetration of "Commercial off the Shelf" SW and HW components across product lines
+ to re-use of software
+ to accelerate development and maintenance
+ to improve containment of product and process complexity and risk
+ to optimize costs of scalable systems

## Working Groups

+ Cross-Standard Lead Working Groups
  + WG-A (Lead Working Group Architecture)
  + WG-A (Lead Working Group Architecture)
  + WG-SEC (Lead Working Group Security)
  + WG-SEC (Lead Working Group Security)
+ Adaptive Platform Working Groups (AP)
  + Adaptive Platform Working Groups (AP)
    + Application start and stop
    + Application health monitoring
    + Application integrity management
    + Mode management
    + Adaptive Platform time synchronization
    + The responsibility for providing the specification of OS interface and execution management in the Adaptive Platform
  + WG-AP-DI (Working Group Adaptive Platform Demonstrator Integration)
  + WG-AP-ST (Working Group Adaptive Platform System Test)
  + WG-AP-PER (Working Group Adaptive Platform Persistency)
  + WG-AP-CCT (Working Group Adaptive Platform Central Coding Team)
  + WG-AP-CLD (Working Group Adaptive Platform Cloud)
+ Classic Platform Working Groups (CP)
  + WG-CP-RTE (Working Group Classic Platform Runtime Environment)
  + WG-CP-MCBD (Working Group Classic Platform Multi core BSW distribution)
    + Motivation is to enable the users to make full- and efficient use of all the micro controller cores.
  + WG-CP-MCL (Working Group Classic Platform Micro controller abstraction layer)
  + WG-CP-LIB (Working Group Classic Platform Libraries)

## Standards

![Standards of AutoSAR](/commons/images/3141911-20230423161231093-368420418.png)

### Acceptance Tests for Classic Platform

AUTOSAR acceptance tests are system tests at both bus level and application level. The AUTOSAR stack is considered as a black box. This means that a provider of such a stack can use these tests to provide initial proof that its implementation complies with the standard.
![Application Interface](/commons/images/3141911-20230425162325755-1152162831.png)

### Application Interface

AUTOSAR standardized a large set of application interfaces in terms of syntax and semantics for the vehicle domains shown in the figure below. 
The application interfaces are released as subset of the classic platform release. Therfore the history of the application interfaces can be found using the search function by the selection of the following parameter:

+ Release: choose the desired release
+ Platform: CP
+ Architecture Element: one of the below shown domains.
  ![AUTOSAR Application Interfaces R22-11](/commons/images/3141911-20230423162200637-1714051464.png)
  The application interface descriptions contain a richness of data standardized by experts of all partners. These standardized interfaces allow software designers and implementers to use them in case of expanding or reusing software components independent of a specific hardware and/or Electronic Control Unit (ECU).

### Classic Platform

The AUTOSAR Classic Platform architecture distinguishes on the highest abstraction level between three software layers which run on a microcontroller: application, runtime environment (RTE) and basic software (BSW).

+ The application software layer is mostly hardware independent.
+ Communication between software components and access to BSW via RTE.
+ The RTE represents the full interface for applications.
+ The BSW is divided in three major layers and complex drivers:
  + Services,
  + ECU (Electronic Control Unit) abstraction,
  + and microcontroller abstraction.
+ Services are divided furthermore into functional groups representing the infrastructure for system, memory and communication services.

![AUTOSAR Classic Release R22-11](/commons/images/3141911-20230423162602228-543776367.png)

![AUTOSAR Hierarchical Structure](/commons/images/3141911-20230424164036880-585885308.png)
In the application layer, the software of AUTOSAR is organized in an independent unit software of AUTOSAR is organized in an independent unit software component(software-component), which encapsulates some or all of the functions and behaviors of automotive electronics, including the realization of specific module functions and corresponding descriptions, but the external ONLY the defined interface is opened, which is called PortPrototypes, and all communications between ECU internal components and actions to obtain other ECU internal components and actions to obtain other ECU resources must be completed by accessing RTE through the interface.
![More specified structure](/commons/images/3141911-20230424164239100-1595103706.png)

1. Software components can communicate with other software components on the same ECU
2. Software components can communicate with other software components located on different ECUs
3. The software component can communicate with the basic software (BSW) that has a port and is located on the same ECU

One essential concept is the virtual functional bus (VFB). This virtual bus decouples the applications from the infrastructure. It communicates via dedicated ports, which means that the communication interfaces of the application software must be mapped to these ports. The VFB handles communication both within the individual ECU and between ECUs. From an application point of view, no detailed knowledge of lower-level technologies or dependencies is required. This supports hardware-independent development and usage of application software.
(The running environment RTE is the implementation of the VFB interface on a specific single ECU, which can be understood as the creation of objects in an object-oriented programming language.)
The AUTOSAR layered architecture is offering all the mechanisms needed for software and hardware independence. It distinguishes between three main software layers which run on a Microcontroller (µC): application layer, runtime environment (RTE), and basic software (BSW).

==`<font face='TIMES' size='5'>`Methodology and Templates`</font>`==

In addition to a software architecture, AUTOSAR introduced a harmonized methodology approach for the development of automotive software. This is mainly driven by the need to improve the collaboration between the different parties involved in today’s automotive projects. 

### Adaptive Platform

The AUTOSAR Adaptive Platform implements the AUTOSAR Runtime for Adaptive Applications (ARA). Two types of interfaces are available, services and APIs. The platform consists of functional clusters which are grouped in services and the Adaptive AUTOSAR Basis.  

Functional clusters...

+ assemble functionalities of the Adaptive Platform
+ define clustering of requirements specification
+ describe behavior of software platform from application and network perspective
+ but, do not constrain the final SW design of the architecture implementing the Adaptive Platform.

In comparison to the AUTOSAR Classic Platform the AUTOSAR Runtime Environment for the Adaptive Platform dynamically links services and clients during runtime.

![AUTOSAR Classic Release R22-11](/commons/images/3141911-20230425162243005-229444589.png)

==`<font face='TIMES' size='5'>` Methodology `</font>`==
AUTOSAR extends the existing Methodology to be able to have a common Methodology approach for both: Classic and Adaptive Platform. The support for distributed, independent, and agile development of functional applications requires a standardized approach on the development methodology. AUTOSAR adaptive methodology involves the standardization of work products and their respective tasks.

+ Work products describe artifacts such as services, applications, machines, and their configuration.
+ The respective tasks define how the work products exchange design information for the activities required which are needed to develop products based on the adaptive platform.

### Foundation

The purpose of the Foundation standard is to enforce interoperability between the AUTOSAR platforms.
Foundation contains common requirements and technical specifications (for example protocols) shared between the AUTOSAR platforms.

![AUTOSAR Foundation Release R22-11](/commons/images/3141911-20230423164436677-1926694094.png)

# Different Position in Product life cycle

|               OEM               | TIER1            | TIER2            |
| :-----------------------------: | :--------------- | :--------------- |
| original equipment manufacturer | Tier 1 suppliers | Tier 2 suppliers |

1. OEMs focus on EE architecture design, application-layer functional design,  methodology and methodology-based SWC design.
2. Systems Engineer, Software Architecture Engineer, Basic Software Engineer, Base-level driver engineer, BSW Protocol Stack Engineer, Complex drive engineer, Integration Engineer.
3. TIER2 DELVES INTO SIMILAR CONTENT TO THE BSW ENGINEERS AT TIER 1, MAINLY FOCUSING ON THE CHIP MCAL AND THE BASIC SOFTWARE PROTOCOL STACK.
4. Tool vendor. The production of many ARXML format configuration files is inseparable from the support of tools, as well as compilation environments and modeling tools.

# Development Phases

1. The approach for developing ECU software was defined by the AUTOSAR organization in its AUTOSAR methodology. It subdivides the development process into actions and standardizes data exchange between development partners with a set of XML files.
2. In system design, the architecture of the application is established. This is accomplished by defining SWCs and their distribution to the ECUs. The network communication is also defined in this step. The result is the System Description – an AUTOSAR XML file, from which a specific ECU Extract of System Description is generated for each ECU.
3. During ECU development, the SWCs are implemented, and the BSW and RTE are configured. During the configuration, the developer defines the basic software contents that are needed for a specific project. This lets the developer optimize the entire ECU software. As a result of the configuration the developer obtains an ECU Configuration Description (AUTOSAR XML file), which is aligned with the ECU Extract of System Description.
4. Code generators are used to generate or modify the Basic Software component of the ECU software based on the ECU Configuration Description. The RTE is also generated to be ECU-specific.
5. The development of the application can be considered to be independent of this process. The interfaces of the SWCs are described in the SWC Descriptions (AUTOSAR XML file). Based on these descriptions, SWCs can be implemented and tested independently of other SWCs. This simplifies integration of application components from the OEM and the Tier1.

# Document naming conventions

EXP: Explaination
MMOD: Meta Model. AUTOSAR metamodel
MOD: Model. Principles of modeling
RS: Requirement Specification.
SRS: Requirement Specification.
SWS: Requirement Specification.
TPS: Requirement Specification.
TR: Requirement Specification.
