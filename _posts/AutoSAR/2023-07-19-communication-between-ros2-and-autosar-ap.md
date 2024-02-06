---
title: Communication between ROS2 and AutoSAR-AP
date: 2023-07-19
categories: [AutoSAR]
tags: [autosar, ros2]     # TAG names should always be lowercase
---
<center><font face='TIMES' size='6'> Communication between ROS2 and AutoSAR-AP </font></center>

[toc]

# Statement

The experiment about the communication between ROS2 and AutoSAR-AP will be divided into following sections:

1. ROS2 Pub ROS2 Sub
2. AP Pub AP Sub
3. ROS2 Pub and AP Sub
4. ROS2 Sub and AP Pub

The underlying logic has been described in the blog: ==[ROS2 and fastdds communication](https://www.cnblogs.com/holylandofdora/p/17338193.html)==

# AutoSAR-AP

## Fusion and Radar Activity

Fusion and radar have different radar activity, Sub/Pub.

![radar activity](/commons/images/3141911-20230511171444257-266641133.png)

## apgen contents

Extract from database and get the config of object

![idl common file from](/commons/images/3141911-20230511194411284-1077504705.png)

```python
common_idl_dir=os.path.join(process_output,'net-bindings','fastdds',"dds_idl") 
# this means idl files in the fusion directory fusion/network-bindings/fastdds/dds_idl is the target common directory
shutil.copytree(os.path.join(self.include,"dds_idl"),common_idl_dir)
self.generateIdl(process_output,common_idl_dir)
self.generateCommonCmake(process_output)
self.generateUserPart(process, instance_specifier, component, process_output)
```

Different Machines and their processes.

![processes of Machines](/commons/images/3141911-20230511200746243-2102452998.png)

ByteArray and different data structures will be generated in idl.

![data structures in STD_CPP_IMPLEMENTATION_DATA_TYPE](/commons/images/3141911-20230511200901112-1755729220.png)

`STD_CPP_IMPLEMENTATION_DATA_TYPE` table contains different definitions of data types.

![definitions of data types](/commons/images/3141911-20230511212530522-1518607771.png)

Manifest file of fusion and radar: `/home/quintin/Documents/AutoSARAP/CAROS35/ARA/Linearx/src/RadarFusionMachine/machines`
![Manifest file of fusion and radar](/commons/images/3141911-20230516162015939-1542574999.png)

# Project Sturcture

cmakelist file of fusion's env.
`/home/quintin/Documents/AutoSARAP/CAROS35/ARA/Linearx/src/RadarFusionMachine/fusion/CMakeLists.txt`

```cmake
if(NOT GEN_PATH)
    set(GEN_PATH ${CMAKE_CURRENT_LIST_DIR})
endif()

set(GEN_SRCS
${GEN_PATH}/net-bindings/fastdds/fastdds_service_mapping-fusion.cpp
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/fastdds_impl_type_bytearray.cxx
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/fastdds_impl_type_radar_field_get_updaterate_in.cxx
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/type_converter_radarobjects.cpp
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/fastdds_radar_common.cxx
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/fastdds_impl_type_radarobjectseventtype.cxx
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/fastdds_impl_type_radarobjects_fieldPubSubTypes.cxx
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/fastdds_impl_type_radar_field_call.cxx
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/fastdds_impl_type_radar_field_return.cxx
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/fastdds_impl_type_radar_field_get_updaterate_result.cxx
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/fastdds_impl_type_radarobjectseventtypePubSubTypes.cxx
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/fastdds_impl_type_radarobjects_fieldfieldnotifiertypePubSubTypes.cxx
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/fastdds_impl_type_radar_field_set_updaterate_inPubSubTypes.cxx
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/fastdds_impl_type_radar_field_set_updaterate_outPubSubTypes.cxx
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/fastdds_impl_type_radar_field_get_updaterate_out.cxx
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/fastdds_impl_type_radarobjectsPubSubTypes.cxx
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/fastdds_impl_type_radar_field_set_updaterate_out.cxx
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/fastdds_impl_type_radar_field_get_updaterate_inPubSubTypes.cxx
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/fastdds_impl_type_radar_field_request.cxx
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/service_desc_radar.cpp
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/fastdds_impl_type_radar_field_requestPubSubTypes.cxx
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/fastdds_impl_type_radar_field_reply.cxx
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/fastdds_impl_type_radarobjects_field.cxx
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/fastdds_impl_type_radar_field_get_updaterate_outPubSubTypes.cxx
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/fastdds_impl_type_radar_field_set_updaterate_in.cxx
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/fastdds_impl_type_radarobjects_fieldfieldnotifiertype.cxx
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/type_converter_radarobjects_field.cpp
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/fastdds_impl_type_radar_field_replyPubSubTypes.cxx
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/fastdds_impl_type_radarobjects.cxx
${GEN_PATH}/net-bindings/fastdds/ara/com/sample/fastdds_impl_type_radar_field_set_updaterate_result.cxx
${GEN_PATH}/net-bindings/fastdds/dds_idl/ara_corePubSubTypes.cxx
${GEN_PATH}/net-bindings/fastdds/dds_idl/ara_core.cxx
${GEN_PATH}/net-bindings/fastdds/dds_idl/dds_rpc.cxx
${GEN_PATH}/net-bindings/fastdds/dds_idl/dds_rpcPubSubTypes.cxx
${GEN_PATH}/net-bindings/fastdds/dds_idl/dds_base_types.cxx
${GEN_PATH}/net-bindings/ara_com_main-fusion.cpp
)

set(GEN_INCS
${GEN_PATH}/includes
${GEN_PATH}/net-bindings/fastdds
)
```

cmakelist file of fusion's main

`/home/quintin/Documents/AutoSARAP/CAROS35/ARA/Linearx/src/RadarFusionMachine/fusion/fusion/CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.15)
get_filename_component(PROJECTNAME ${CMAKE_CURRENT_SOURCE_DIR} NAME)
project(${PROJECTNAME} VERSION 1.0.0 LANGUAGES CXX)

include(GNUInstallDirs)

if(NOT APP_NAME)
    set(APP_NAME "fusion")
endif()
if(NOT PROC_NAME)
    set(PROC_NAME "fusion")
endif()

# Use the currently recommended way to set a default value for CMAKE_INSTALL_PREFIX; this
# does have limitations (see CMake issue 21242), but they don't matter much for us here.
set(CMAKE_INSTALL_PREFIX ${CMAKE_CURRENT_LIST_DIR}/../../../..) #if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)

SET(CMAKE_BUILD_TYPE DEBUG)
set(CMAKE_PREFIX_PATH ${CMAKE_CURRENT_LIST_DIR}/../../../../lib/cmake)
message("export LD_LIBRARY_PATH=${CMAKE_CURRENT_LIST_DIR}/../../../../lib:$LD_LIBRARY_PATH")
set(eProsima ${CMAKE_CURRENT_LIST_DIR}/../../../../3rd-party/eProsima)
set(fastrtps_DIR ${eProsima}/Fast-DDS/share/fastrtps/cmake)
set(fastcdr_DIR ${eProsima}/Fast-CDR/lib/cmake/fastcdr)
set(foonathan_memory_DIR ${eProsima}/foonathan_memory_vendor/lib/foonathan_memory/cmake)
find_package(Threads REQUIRED)
find_package(ara-com REQUIRED)
find_package(ara-core REQUIRED)
find_package(ara-log REQUIRED)
find_package(ara-per REQUIRED)
find_package(ara-exec-execution-client REQUIRED)
find_package(fastrtps REQUIRED)
set(COMMUNICATION_LIBRARY ara_fastddsbinding)
add_compile_definitions(HAS_FASTDDS_BINDING)

include(${CMAKE_CURRENT_LIST_DIR}/../CMakeLists.txt)

AUX_SOURCE_DIRECTORY(src   APPLICATION_SOURCES) #add all source file in directory src

add_executable(${CMAKE_PROJECT_NAME} ${APPLICATION_SOURCES} ${GEN_SRCS})

target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE
include
${fastrtps_INCLUDE_DIR}
${fastcdr_INCLUDE_DIR}
${GEN_INCS}
)
link_directories(
${fastrtps_LIB_DIR}
${fastcdr_LIB_DIR}
)

target_link_libraries(${CMAKE_PROJECT_NAME}
${COMMUNICATION_LIBRARY}
ara_com 
ara::log 
ara::exec_execution_client 
ara::core

ara::ara_fileaccess
ara::ara_kvsparser
ara::ara_kvstype
ara::ara_keyvaluestorage
ara::ara_manifestaccess
ara::ara_per_utility
ara::ara_updateper
ara::ara_redundancy

fastrtps
fastcdr)

# install bin
install( TARGETS ${CMAKE_PROJECT_NAME}
         DESTINATION opt/fusion/bin
)

# MANIFEST
install(FILES ${CMAKE_CURRENT_LIST_DIR}/../processes/fusion_MANIFEST.json
    DESTINATION opt/fusion/etc RENAME MANIFEST.json)

# service_instance_manifest
install(FILES ${CMAKE_CURRENT_LIST_DIR}/../processes/fusion_SI_MANIFEST.json
    DESTINATION opt/fusion/etc RENAME service_instance_manifest.json)
```

# Function and header file call relationships

All radar activities are implemented in `/home/quintin/Documents/AutoSARAP/CAROS35/ARA/Linearx/src/RadarFusionMachine/fusion/fusion/src/main_fusion.cpp` and `/home/quintin/Documents/AutoSARAP/CAROS35/ARA/Linearx/src/RadarFusionMachine/fusion/fusion/src/radar_activity.cpp`

![radar_activity.h](/commons/images/3141911-20230515162341614-1309820635.png)

Here are contains-relationships in the `radar_activity.h`

```
radar_activity.h
    radar_proxy.h
        ara/com/internal/proxy/ara_proxy_base.h
            proxy_base.h
            proxy_binding_base.h
        radar_common.h
            impl_type_radarobjects.h
                impl_type_bytearray.h
            impl_type_radarobjects_field.h
                impl_type_radarobjects_field.h
```

## radar_proxy in radar_activity

`radar_proxy.h` in `radar_activity.h` defines fields and events in proxy namespace.

![radar_proxy.h](/commons/images/3141911-20230515162915389-1652900118.png)

## radar_common in radar_activity::radar_proxy

`radar_common.h` defines data structures from radarobject
![radar_common.h](/commons/images/3141911-20230515163400722-467797640.png)

`radar_common.h` has the almost same data structure that in `ara/com/sample/impl_type_radarobjects_field.h` and `ara/com/sample/impl_type_radarobjects.h`

fusion is the client-end of the proxy, and there are two different communication method: `event` and `field`.

```cpp
//struct RadarObjects
struct RadarObjects_field
{
    bool active;
    ::ByteArray objectVector;

    using IsEnumerableTag = void;
    template <typename F>
    void enumerate(F& fun)
    {
        fun(this->active);
        fun(this->objectVector);
    }
};
```

![radarObject and radarObject_field](/commons/images/3141911-20230515165007842-1352972317.png)

## how to register and use the fastdds service in fusion(case is the same when it happens in radar)

the entry of the register of proxy service in `fusion`.

![main_fusion.cpp](/commons/images/3141911-20230529172226204-1721926695.png)

1. init in the main fusion.
2. fusion activities.

![init_service](/commons/images/3141911-20230529172427192-83929593.png)

`RPort` means receive. Proxy
`PPort` means provide. Skeleton

![radaractivity](/commons/images/3141911-20230529173545739-928170143.png)

`Proxy::StartFindService` function: Findd service by `ServiceID` defined in original Ap_model database.

![StartFindService function](/commons/images/3141911-20230529174405889-641421170.png)

==`StartFindServiceById` funstion in StartFindService, which return DoStartfinServiceById.==

![StartFindServiceById function](/commons/images/3141911-20230529174902708-1860671317.png)

==The init function of fastdds in ara com main in networkbindings==

![Init function of Findservice](/commons/images/3141911-20230529175311881-1797799774.png)

Registry function in networkbindings, here, the program get the manifest.
and the contents will be stored into the manifest in class `Runtime`.

![registry function in networkbindings](/commons/images/3141911-20230529175517267-1011648998.png)

==`DoStartfinServiceById` function returned by StartFindService==
and the handler will be returned into function `Startfindservice` as `result`.
Finally, the result will be returned back in the `init` function in the fusion.

![DoStartfinServiceById function ](/commons/images/3141911-20230529180527235-973272931.png)

# files in the net-bindings

In `fastdds/ara/com/sample` are the files related to the definition of some data types, such as radar_object and radar_field_object.

## type_info_proxy_impl_radar.h

`type_info_proxy_impl_radar.h` contains two different events: `breakEvent` and `parkingBrakeEvent`, which are both `event` Type, a notifier of the wholeall field.

![type_info_proxy_impl_radar.h](/commons/images/3141911-20230515153119260-1800442739.png)

For example, there are 2 events in total in the `radar_activity.cpp`.

`act()` function has two events: `brakeEvent` and `parkingBrakeEvent` in `rardar_activity.cpp`
![radar_activity.cpp](/commons/images/3141911-20230515162452110-1816165541.png)

## fastdds_impl_type_radarobjecteventtypePubSubTypes.h

Inheritance relationships in the files. Radar object events are defined in continuity relationship. `fastdds_impl_type_radarobjecteventtypePubSubTypes.h`

![fastdds_impl_type_radarobjecteventtypePubSubTypes.h](/commons/images/3141911-20230515153628776-876188088.png)

RadarObjectsEvenTypePubSubType in the `cxx` field.

![fastdds_impl_type_radarobjecteventtypePubSubTypes.cxx](/commons/images/3141911-20230515153704988-341533458.png)

RadarObjectEventType is used in the `fastdds_impl_type_radarobjecteventtypePubSubTypes`, and defined in the `fastdds_impl_type_radarobjectseventtype.h`

![fastdds_impl_type_radarobjectseventtype.h](/commons/images/3141911-20230515154031261-366052535.png)

## fastdds_impl_type_radarobject.h

radarobjects_type been included in the RadarObjectEventType.h

![fastdds_impl_type_radarobject.h](/commons/images/3141911-20230515154519181-278880303.png)

## fastdds_impl_bytearray.h in dds_idl

fastdds bytearrat that been included in the radarobjects_type

![dds_idl.fastdds_impl_bytearray](/commons/images/3141911-20230515154921199-1081403615.png)

'dds_idl.dds_base_types.h' includes all needed bascic types.

![dds_idl.dds_base_types.h](/commons/images/3141911-20230515155027405-2133171544.png)

# Modification of the idl file in the ros2

## Generation process of idl in ap_gen.py

AP settings are from PH3B.dat, which are showed in Part.2.2: AutoSAR-AP.apgen contents.

![machines in ap_model](/commons/images/3141911-20230530094436461-1336678248.png)

machines structure extracted from ap_model
![fusion and radar structure in machines](/commons/images/3141911-20230530094536056-1342831819.png)

processes in different machines.
![process standard name in machines ](/commons/images/3141911-20230530094604892-134611113.png)

radar process's components are from location `apd/AdaptiveApplicationSwComponentTypes/radar`.

![components location settings in radar process](/commons/images/3141911-20230530094643938-309051330.png)

`fqn` is where components from

![fqn is where components from](/commons/images/3141911-20230530095522859-851534864.png)

generating idl files command args.

![args composition](/commons/images/3141911-20230530095714361-1878321439.png)

common idl dir is where templates idl from: `net-bindings/fastdds/dds_idl/`
process output is where to put different files into.

![generation command using fastddsgen](/commons/images/3141911-20230530100324066-1196286445.png)

`generateIdl` function inclusion.

![generateIdl function](/commons/images/3141911-20230530100415005-254919109.png)

## Modification of generated files afater ap_gen.

1. Change the topic name in the service description radar file.
   this step can also be realized by using `-rostype`.
   ![service_desc_radar.h](/commons/images/3141911-20230530101033439-626560019.png)
2. Changes in type name, it can also be realized in the step1, using option `rostype`.
   ![radarobject in the PubSub class](/commons/images/3141911-20230530101637206-1653431141.png)
3. type name should be consistant with dds_ type type_info_proxy_impl_radar.
   ![type_info_proxy_impl_radar](/commons/images/3141911-20230530102101865-910072754.png)
4. Typename can be referred in the implememt file: `/ARA/Linearx/src/RadarFusionMachine/fusion/net-bindings/fastdds/ara/com/sample/type_info_proxy_impl_radar.h` Other files such as field request and reply cpp have defined the typename of fieldrequest and fieldreply.
   ![type_info_proxy_impl_radar.h](/commons/images/3141911-20230602153432382-1119460195.png)

## idl files in the ROS End.

Some examples of mistakes.

1. Currently only .idl files with 1 (a message), 2 (a service) and 3 (an action) structures are supported /home/quintin/Documents/ROS_ws/src/apidl_interfaces/msg/fastdds_impl_type_bytearray.idl
   Error processing idl file: /home/quintin/Documents/ROS_ws/src/apidl_interfaces/msg/fastdds_impl_type_bytearray.idl
   ![error1](/commons/images/3141911-20230530104454957-184327666.png)
2. error not exist file and sturct.
   ![error file not exist](/commons/images/3141911-20230530104759673-1996019303.png)
3. error types of idl file.
   ![error types of idl file](/commons/images/3141911-20230530110830302-1393215345.png)

Rectify arrangement of idl file in ros interfaces.
![final correct type in ros interfaces](/commons/images/3141911-20230530111127511-1088561316.png)

All the reasons boil down to the following, the module inclusion relationship of the idl interfaces of ros2 corresponds to the folder, moreover, the naming is consistent with the folder content, and the underlying struct naming is consistent with the idl file.

(Do not nest more than 4 layers!)

# ROS2 Pub and ROS2 Sub

1. construct the message idl file.
   Whole process is the same as which in the last step.
2. Change the topic name, which is in consistency with the `AP` END.
3. build and run.

Success in ros2 Pub/Sub.
![success ros2 Pub and Sub](/commons/images/3141911-20230531154006009-1933784557.png)

Hunan: Find the resources where different packages are from: githubusercontent.com/ros2/ros2/humble/ros2.repos

Install ROS2 humble and find out its contents version:

eg. find the version of `Fast-DDS`, and its version is 2.6.x.
![ros2 repos](/commons/images/3141911-20230601165451769-688895169.png)

# AP Pub and AP Sub

After Modification, and annotating event and field except breakEvent, the whole Process is like this:
![AP Pub and AP Sub](/commons/images/3141911-20230602164633086-862756384.png)

And the radar and fusion process before being annotated is like this:

1. radar_machines_provide

```txt
/bin/radar 
[86251.176904]~DLT~81366~INFO     ~FIFO /tmp/dlt cannot be opened. Retrying later...
2023/05/31 02:02:42.829425  862511770 001 ECU1 ---- INTM log info V 2 [Console output enabled appId:]
2023/05/31 02:02:42.829478  862511771 002 ECU1 ---- INTM log error V 4 [DLT back-end init finalization failure appId: log level:6 status:-1]
2023/05/31 02:02:42.830435  862511780 000 ECU1 ---- EXCL log error V 2 [ExecutionClient::ReportExecutionState(): Bad file descriptor]
2023/05/31 02:02:42.830490  862511781 000 ECU1 ---- DFLT log info V 1 [Radar: configure e2e protection]
Error: ./etc/e2e_dataid_mapping.json: cannot open file
2023/05/31 02:02:42.830679  862511783 000 ECU1 ---- E2ES log error V 2 [Failed to configure status handler due to:  Unable to parse empty property tree.]
2023/05/31 02:02:42.830712  862511783 001 ECU1 ---- DFLT log info V 2 [Radar: e2e configuration  failed]
2023/05/31 02:02:42.830761  862511783 002 ECU1 ---- DFLT log info V 1 [Ok, let's start doing fancy stuff...]
2023/05/31 02:02:42.830922  862511785 000 ECU1 ---- CTX4 log info V 2 [Port In Executable Ref: radar/radar/radar_PPort]
2023/05/31 02:02:42.832215  862511798 001 ECU1 ---- CTX4 log info V 2 [Service Instance offered: DDS:19]
2023/05/31 02:02:42.832399  862511800 002 ECU1 ---- CTX4 log debug V 2 [object address 0x00007ffbfc1fcd10]

//init() start
2023/05/31 02:02:42.832526  862511801 000 ECU1 ---- CTX3 log debug V 1 [enter init()]

//act() start
2023/05/31 02:02:42.838938  862511865 001 ECU1 ---- CTX3 log info V 1 [radar active]
2023/05/31 02:02:42.839027  862511866 002 ECU1 ---- CTX3 log info V 1 [brakeEvent sent]
2023/05/31 02:02:42.839069  862511867 003 ECU1 ---- CTX3 log info V 1 [parkingBrakeEvent sent]
2023/05/31 02:02:42.839106  862511867 004 ECU1 ---- CTX3 log info V 1 [UpdateRate notify ]
//act()end

2023/05/31 02:02:43.239335  862515869 005 ECU1 ---- CTX3 log info V 1 [radar active]
2023/05/31 02:02:43.239550  862515871 006 ECU1 ---- CTX3 log info V 1 [brakeEvent sent]
2023/05/31 02:02:43.239564  862515871 007 ECU1 ---- CTX3 log info V 1 [parkingBrakeEvent sent]

2023/05/31 02:02:43.239578  862515872 008 ECU1 ---- CTX3 log info V 1 [UpdateRate notify ]
2023/05/31 02:02:43.640458  862519880 009 ECU1 ---- CTX3 log info V 1 [radar active]
2023/05/31 02:02:43.640596  862519882 010 ECU1 ---- CTX3 log info V 1 [brakeEvent sent]
2023/05/31 02:02:43.640627  862519882 011 ECU1 ---- CTX3 log info V 1 [parkingBrakeEvent sent]
```

2. fusion_machiens_receive:

```txt
[86274.613358]~DLT~81548~INFO     ~FIFO /tmp/dlt cannot be opened. Retrying later...
2023/05/31 02:03:06.265935  862746135 001 ECU1 ---- INTM log info V 2 [Console output enabled appId:]
2023/05/31 02:03:06.266012  862746136 002 ECU1 ---- INTM log error V 4 [DLT back-end init finalization failure appId: log level:6 status:-1]
2023/05/31 02:03:06.266140  862746137 000 ECU1 ---- EXCL log error V 2 [ExecutionClient::ReportExecutionState(): Bad file descriptor]
2023/05/31 02:03:06.266184  862746138 000 ECU1 ---- DFLT log info V 1 [Fusion: configure e2e protection]
Error: ./etc/e2e_dataid_mapping.json: cannot open file
2023/05/31 02:03:06.266349  862746139 000 ECU1 ---- E2ES log error V 2 [Failed to configure status handler due to:  Unable to parse empty property tree.]
2023/05/31 02:03:06.266368  862746140 001 ECU1 ---- DFLT log info V 2 [Fusion: e2e configuration  failed]
2023/05/31 02:03:06.266388  862746140 002 ECU1 ---- DFLT log info V 1 [Ok, let's start doing fancy stuff...]
2023/05/31 02:03:06.266504  862746141 000 ECU1 ---- FACT log debug V 2 [object address 0x00007f6aab3fb960]
// init() start
2023/05/31 02:03:06.266543  862746141 001 ECU1 ---- FACT log info V 1 [init() enter]
2023/05/31 02:03:06.266556  862746141 002 ECU1 ---- FACT log info V 2 [Port In Executable Ref: fusion/fusion/radar_RPort]
2023/05/31 02:03:06.266757  862746143 003 ECU1 ---- FACT log info V 2 [Searching for Service Instance: DDS:19]
2023/05/31 02:03:06.283500  862746311 004 ECU1 ---- FACT log info V 3 [Instance  DDS:19  is available]
2023/05/31 02:03:06.285755  862746333 005 ECU1 ---- FACT log info V 2 [Created proxy from handle with instance:  DDS:19]
2023/05/31 02:03:06.285813  862746334 006 ECU1 ---- FACT log info V 2 [Check handle::operator==:  1]
2023/05/31 02:03:06.285823  862746334 007 ECU1 ---- FACT log info V 2 [Check handle::operator<:  0]
2023/05/31 02:03:06.285896  862746335 008 ECU1 ---- FACT log info V 1 [init() exit]
// init() end

//act() start
2023/05/31 02:03:06.285933  862746335 009 ECU1 ---- FACT log info V 1 [radar alive]
2023/05/31 02:03:06.285942  862746335 010 ECU1 ---- FACT log info V 1 [not subscribed to brakeEvent yet]
2023/05/31 02:03:06.286169  862746338 011 ECU1 ---- FACT log info V 1 [brakeEvent Callback registered.]
2023/05/31 02:03:06.286204  862746338 012 ECU1 ---- FACT log info V 1 [brakeEvent subscription complete]
// register brakeEvent

2023/05/31 02:03:06.286213  862746338 013 ECU1 ---- FACT log info V 1 [not subscribed to parkingBrakeEvent yet]
2023/05/31 02:03:06.286333  862746339 014 ECU1 ---- FACT log info V 1 [parkingBrakeEvent Callback registered.]
2023/05/31 02:03:06.286364  862746339 015 ECU1 ---- FACT log info V 1 [parkingBrakeEvent subscription complete]
// register parkingBreakEvent

2023/05/31 02:03:06.286374  862746340 016 ECU1 ---- FACT log info V 1 [Subscribe to UpdateRate Field]
2023/05/31 02:03:06.286386  862746340 017 ECU1 ---- FACT log info V 1 [UpdateRateSubscription()]
2023/05/31 02:03:06.286548  862746341 018 ECU1 ---- FACT log info V 1 [UpdateRate Callback registered.]
2023/05/31 02:03:06.286579  862746342 019 ECU1 ---- FACT log info V 1 [done]
//register field which named UpdateRate, and it contains event as notifier and method as message transfers

2023/05/31 02:03:06.286587  862746342 020 ECU1 ---- FACT log info V 2 [FIELDS:  Trying to get values for UpdateRate]
// in the function fieldGetter(), which will get 



2023/05/31 02:03:06.440555  862747881 021 ECU1 ---- FACT log verbose V 2 [FIELDS:  Callback: UpdateRate Field ]
2023/05/31 02:03:06.496689  862748443 022 ECU1 ---- FACT log verbose V 2 [FIELDS:  Callback: UpdateRate Field ]
2023/05/31 02:03:06.897597  862752452 023 ECU1 ---- FACT log verbose V 2 [FIELDS:  Callback: UpdateRate Field ]
2023/05/31 02:03:07.299109  862756467 024 ECU1 ---- FACT log verbose V 2 [FIELDS:  Callback: UpdateRate Field ]
2023/05/31 02:03:07.700761  862760483 025 ECU1 ---- FACT log verbose V 2 [FIELDS:  Callback: UpdateRate Field ]
2023/05/31 02:03:08.101647  862764492 026 ECU1 ---- FACT log verbose V 2 [FIELDS:  Callback: UpdateRate Field ]
^C

// annotate the ParkingEvent and Field:
quintin@ubuntu:~/Documents/AutoSARAP/CAROS35/ARA/Linearx/opt/fusion$ ./bin/fusion 
[112545.337546]~DLT~110104~INFO     ~FIFO /tmp/dlt cannot be opened. Retrying later...
2023/05/31 23:40:13.233283 1125453377 001 ECU1 ---- INTM log info V 2 [Console output enabled appId:]
2023/05/31 23:40:13.233351 1125453378 002 ECU1 ---- INTM log error V 4 [DLT back-end init finalization failure appId: log level:6 status:-1]
2023/05/31 23:40:13.233500 1125453380 000 ECU1 ---- EXCL log error V 2 [ExecutionClient::ReportExecutionState(): Bad file descriptor]
2023/05/31 23:40:13.233556 1125453380 000 ECU1 ---- DFLT log info V 1 [Fusion: configure e2e protection]
Error: ./etc/e2e_dataid_mapping.json: cannot open file
2023/05/31 23:40:13.233811 1125453383 000 ECU1 ---- E2ES log error V 2 [Failed to configure status handler due to:  Unable to parse empty property tree.]
2023/05/31 23:40:13.233876 1125453383 001 ECU1 ---- DFLT log info V 2 [Fusion: e2e configuration  failed]
2023/05/31 23:40:13.233905 1125453384 002 ECU1 ---- DFLT log info V 1 [Ok, let's start doing fancy stuff...]
2023/05/31 23:40:13.234069 1125453385 000 ECU1 ---- FACT log debug V 2 [object address 0x00007fec2b1fb960]
2023/05/31 23:40:13.234104 1125453386 001 ECU1 ---- FACT log info V 1 [init() enter]
2023/05/31 23:40:13.234125 1125453386 002 ECU1 ---- FACT log info V 2 [Port In Executable Ref: fusion/fusion/radar_RPort]
2023/05/31 23:40:13.235332 1125453398 003 ECU1 ---- FACT log info V 2 [Searching for Service Instance: DDS:19]
2023/05/31 23:40:13.242687 1125453471 004 ECU1 ---- FACT log info V 3 [Instance  DDS:19  is available]
2023/05/31 23:40:13.245571 1125453500 005 ECU1 ---- FACT log info V 2 [Created proxy from handle with instance:  DDS:19]
2023/05/31 23:40:13.245618 1125453501 006 ECU1 ---- FACT log info V 2 [Check handle::operator==:  1]
2023/05/31 23:40:13.245624 1125453501 007 ECU1 ---- FACT log info V 2 [Check handle::operator<:  0]
2023/05/31 23:40:13.245907 1125453504 008 ECU1 ---- FACT log info V 1 [init() exit]
2023/05/31 23:40:13.245943 1125453504 009 ECU1 ---- FACT log info V 1 [radar alive]
2023/05/31 23:40:13.245961 1125453504 010 ECU1 ---- FACT log info V 1 [not subscribed to brakeEvent yet]
2023/05/31 23:40:13.246375 1125453508 011 ECU1 ---- FACT log info V 1 [brakeEvent Callback registered.]
2023/05/31 23:40:13.246402 1125453509 012 ECU1 ---- FACT log info V 1 [brakeEvent subscription complete]
2023/05/31 23:40:13.647011 1125457515 013 ECU1 ---- FACT log info V 1 [radar alive]
2023/05/31 23:40:13.647079 1125457515 014 ECU1 ---- FACT log verbose V 2 [brakeEvent E2E state: not ok]
2023/05/31 23:40:13.647153 1125457516 015 ECU1 ---- FACT log verbose V 1 [recive brakeEvent message]
2023/05/31 23:40:14.048081 1125461525 016 ECU1 ---- FACT log info V 1 [radar alive]
2023/05/31 23:40:14.048192 1125461527 017 ECU1 ---- FACT log verbose V 2 [brakeEvent E2E state: not ok]
2023/05/31 23:40:14.048234 1125461527 018 ECU1 ---- FACT log verbose V 1 [recive brakeEvent message]
```

# Function of the setpartition in the fastdds

## The Calling relation of Setpartition.

## concept of partition in fastdds.

Partitions introduce a logical entity isolation level concept inside the physical isolation induced by a Domain.

1. Unlike Domain and Topic, Partitions acan be chanted dynamically during the life cycle of the endpoint with little cost.
2. Modifyying the Partition membership of endpoints will trigger the announcement of the new Qos configuration.
3. Unlike Domain and Topic,  and endpoint can belong to several Partitions at the same time. For certain data to be shared over different Topics, there must be a different Publisher for each Topic, each of them sharing its own his tory of changes. On the other hand, a single Publisher can share the same data over different Partitions using a single topic datga change, thus reducing network overload.

A system with the following Partition configuration.
![Partition configuration](/commons/images/3141911-20230602142017831-456643542.png)

Match the Partitions depicted on the following table.
![Matching table in the example](/commons/images/3141911-20230602142046398-1353965604.png)

The communication matrix for the given example.
![communication matrix](/commons/images/3141911-20230602142100296-2032285665.png)
a short example in C++.

```cpp
PublisherQos pub_11_qos;
pub_11_qos.partition().push_back("Partition_1");
pub_11_qos.partition().push_back("Partition_2");

PublisherQos pub_12_qos;
pub_12_qos.partition().push_back("*");

PublisherQos pub_21_qos;
//No partitions defined for pub_21

PublisherQos pub_22_qos;
pub_22_qos.partition().push_back("Partition*");

SubscriberQos subs_31_qos;
subs_31_qos.partition().push_back("Partition_1");

SubscriberQos subs_32_qos;
subs_32_qos.partition().push_back("Partition_2");

SubscriberQos subs_33_qos;
subs_33_qos.partition().push_back("Partition_3");

SubscriberQos subs_34_qos;
//No partitions defined for subs_34
```

## Implement in Linearx AutoSar

> ara::com::internal::dds::CreateSubscriber
> ara::com::internal::dds::CreatePublisher
> ara::com::internal::dds::SetPartition

fastdds's implement in LinearX:
`/ARA/Linearx/include/ara/com/internal/fastdds/fastdds_domain_participant.h`
`/ARA/Linearx/include/ara/com/internal/dds_idl/domain_participant.h`

`/ARA/ara-api/com/src/libara_fastddsbinding/fastdds_domain_participant.cpp`
`/ARA/ara-api/com/src/libara_ddsidlbinding/domain_participant.cpp`

`fastdds_domain_participant.h`
![fastdds_domain_participant.h](/commons/images/3141911-20230602105313489-1441596746.png)

`domain_participant.cpp`
![domain_participant.cpp](/commons/images/3141911-20230602103929172-147483881.png)

Annotate the SetPartition function in CreatePublisher() and CreateSubscriber().
![CreatePublisher function](/commons/images/3141911-20230602103236213-513381481.png)

The implement of function SetPartition:
make a vector and pushback newName to the vector.
![SetPartition function](/commons/images/3141911-20230602102727742-1143825798.png)

# ROS2 Pub and AP Sub

Where AP changes?

1. Every topic should be replaced using ros2 type form.
2. Every type name should be replaced using ros2 type form, including field and notifiers.

## Topic(Event,field) should be consisitent in internal AP communication.

error messages because of incompatible typename:
![error messages because of inconsistent typename](/commons/images/3141911-20230531160623258-1800562670.png)

## Where topic is registered by fastdds.

Find where the topic is been registered, Subscribe(3) function in act() function:
![Subscribe(3) function in act() function](/commons/images/3141911-20230601200232646-1394229560.png)

GetTopic function in Subscribe() function:
![GetTopic function in Subscribe() function](/commons/images/3141911-20230602135253337-941002611.png)

key is the name of topic:
![key is the name of topic](/commons/images/3141911-20230601201259837-64031536.png)

## ROS2 talker and AP fusion(subscriber)

warning: find the topic but incompatible Qos, no messages will be sent to it.
![incompatible Qos cause no message transfer](/commons/images/3141911-20230602170617907-847863750.png)

Check the qos configuration of the ROS2 END:
ROS2 talker End
![qos config of the ROS2 talker End](/commons/images/3141911-20230602172748744-1617512139.png)

ROS2 listener End
![qos config of the ROS2 listener End](/commons/images/3141911-20230602180609282-1452831085.png)

### THe default Qos definition of the AP sub and pub

`std::shared_ptr<DataReaderType> Subscriber::CreateDataReader` function in `fastdds_subcriber.h`

```cpp
dataReaderQos.durability().kind = eprosima::fastdds::dds::TRANSIENT_LOCAL_DURABILITY_QOS;
dataReaderQos.reliability().kind = eprosima::fastdds::dds::RELIABLE_RELIABILITY_QOS;
dataReaderQos.endpoint().history_memory_policy = eprosima::fastrtps::rtps::DYNAMIC_RESERVE_MEMORY_MODE;
```

![CreateDataReader](/commons/images/3141911-20230616155624772-1611044533.png)

### QOS Compatibility Rule

[QOS rule of QOS](https://fast-dds.docs.eprosima.com/en/latest/fastdds/dds_layer/core/policy/standardQosPolicies.html)

[DurabilityQosPolicy_kind](https://fast-dds.docs.eprosima.com/en/latest/fastdds/dds_layer/core/policy/standardQosPolicies.html#durabilitykind)

According to web search results, you can change the QoS policy in a ROS 2 cpp file by creating a QoS object with the desired policy values and passing it to the constructor of the publisher or subscriber1. For example, if you want to create a publisher with a reliable reliability policy and a keep last history policy with a depth of 10, you can do something like this:

```cpp
#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);
  auto node = rclcpp::Node::make_shared("talker");

  // Create a QoS object with reliable reliability and keep last history with depth 10
  rmw_qos_profile_t qos_profile = rmw_qos_profile_default;
  qos_profile.reliability = RMW_QOS_POLICY_RELIABILITY_RELIABLE;
  qos_profile.history = RMW_QOS_POLICY_HISTORY_KEEP_LAST;
  qos_profile.depth = 10;
  auto qos = rclcpp::QoS(rclcpp::QoSInitialization(qos_profile.history, qos_profile.depth), qos_profile);

  // Create a publisher with the QoS object
  auto pub = node->create_publisher<std_msgs::msg::String>("chatter", qos);

  // Publish messages as usual
  auto msg = std_msgs::msg::String();
  msg.data = "Hello world";
  pub->publish(msg);

  rclcpp::spin_some(node);
  rclcpp::shutdown();
  return 0;
}
```

You can also use some predefined QoS profiles that are suitable for common use cases, such as sensor data or services2. For example, if you want to use the sensor data profile, you can do something like this:

```cpp
#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);
  auto node = rclcpp::Node::make_shared("talker");

  // Use the sensor data profile
  rmw_qos_profile_t qos_profile = rmw_qos_profile_sensor_data;
  auto qos = rclcpp::QoS(rclcpp::QoSInitialization(qos_profile.history, qos_profile.depth), qos_profile);

  // Create a publisher with the QoS object
  auto pub = node->create_publisher<std_msgs::msg::String>("chatter", qos);

  // Publish messages as usual
  auto msg = std_msgs::msg::String();
  msg.data = "Hello world";
  pub->publish(msg);

  rclcpp::spin_some(node);
  rclcpp::shutdown();
  return 0;
}
```

You can find more information about how to set QoS policies for ROS 2 nodes in the ROS documentation32.

To maintain the compatibility between DurabilityQosPolicy in DataReaders and DataWriters when they have different kind values, the DataWriter kind must be higher or equal to the DataReader kind. And the order between the different kinds is:

VOLATILE_DURABILITY_QOS < TRANSIENT_LOCAL_DURABILITY_QOS < TRANSIENT_DURABILITY_QOS < PERSISTENT_DURABILITY_QOS

Change the Qos Rule of ROS

Success!

# ROS2 Sub and AP Pub

Success!
