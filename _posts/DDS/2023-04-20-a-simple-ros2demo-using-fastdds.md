---
title: A simple ROS2 demo using Fastdds
date: 2023-04-20
categories: [DDS]
tags: [dds, ros2]     # TAG names should always be lowercase
---
[TOC]

# Statement
Here I will do a simple experment about the Publisher/Subscriber model in BOTH FastDDS and ROS2(humble, latest), as well as cross communication between them.

And my experiment will be divided into the following sections:
1. ROS2 Pub/Sub model.
2. FastDDS Pub/Sub model.
3. FastDDS Pub / ROS2 Sub model.
4. ROS2 Pub / Fast DDS Sub.

And I will discuss two different way to use fastddsgen.
+ Common: fastddsgen HelloWorld.idl
+ ROS2: fastddsgen HelloWorld.idl -typeros2

# Prerequisites in ROS2/FastDDS
Here are some basic concept to comprehend first.
+ Installation of ROS2/FastDDS
  + [Installation and basic deployment in ROS2 Pub/Sub model](https://docs.ros.org/en/humble/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Cpp-Publisher-And-Subscriber.html)
  + [Installation and basic deployment in FastDDS Pub/Sub model](https://fast-dds.docs.eprosima.com/en/latest/fastdds/getting_started/simple_app/simple_app.html)
+ In ROS2 idl method to make a Pub/Sub model, start from pub/sub submodel method, and add the msg/srv files to this subtle model, and finally change it into a intact idl Pub/Sub model.

## ROS2 prep
### pub/sub subtle model
1. Make an ROS2 workspace, and navigate to `src`, to create the directory you want, using
> ros2 pkg create --build-type ament_cmake ==YOURDIR==
2. Navigate to the lowest level src directory in ==YOURDIR==, and write the pub/sub node.
![project structure](/commons/images/3141911-20230420142323486-587877158.png)
3. add dependencies in the `package.xml` and `CMakeList.txt`. Why and how?
    + When create the project using `ros pkg create ...`, the structure of the project contains the `cmake and xml` configuration of the whole project, and any dependencies included in the .CXX file need to be added into them.
    + For example, in file `publisher_member_function.cpp` and `subscriber_member_function.cpp`contains two C++ headers in the ROS 2 system. 
      ```cpp
      #include "rclcpp/rclcpp.hpp"
      #include "std_msgs/msg/string.hpp"
      ```
      so these two dependencies should be added into both .XML and CMakeList
      ```xml
      <!-- after ament_cmake -->
      <depend>rclcpp</depend>
      <depend>std_msgs</depend>
      ```

      ```cmake
      # below the find_package(ament_cmake REQUIRED)
      find_package(rclcpp REQUIRED)
      find_package(std_msgs REQUIRED)

      add_executable(talker src/publisher_member_function.cpp)
      ament_target_dependencies(talker rclcpp std_msgs)
      add_executable(listener src/subscriber_member_function.cpp)
      ament_target_dependencies(listener rclcpp std_msgs)

      # add this section so ros2 run talker can find its executable
      install(TARGETS
      talker
      DESTINATION lib/${PROJECT_NAME})
      # or just do like this, which can include both two
      install(TARGETS
      talker
      listener
      DESTINATION lib/${PROJECT_NAME})
      ```
4. Build and run of your project:  After above operation, you have prepared your project, then you need to build and run it in the ROSWORKSPACE folder layer.
    + run `rosdep` in the root of your workspace to check for missing dependencies before building:
    > rosdep install -i --from-path src --rosdistro humble -y
    + build new packages using command:
    > colcon build --packages-select ==YOURDIR==
    + remember to source your setup file before you want to run your project in a new shell.
    > source ./install/setup.bash
    + finally, run it
    > ros run ==YOURDIR== talker/listener
### pub/sub msg srv model
The project before clarify the basic usage of ros2 to create a pub/sub model in ROS2. Then expand it to a model using msg/srv file.
***
***What are the differences?***
Creating custom .msg and .srv files in their own package, here is the folder `msg` and `srv`.
![msg/srv project structure](/commons/images/3141911-20230420152530814-1692877424.png)
***
1. The definition of the msg and srv files.
    msg/Num.msg
    ```txt
    int64 num
    ```
    msg/Sphere.msg
    ```txt
    geometry_msgs/Point center
    float64 radius
    ```
    srv/AddThreeInts.srv
    ```txt
    int64 a
    int64 b
    int64 c
    ---
    int64 sum
    ```
    What does the srv file stand for?
    This is custom service that requests three integers named a, b, and c, and responds with an integer called sum.
2. Changes in CMakeLists.txt and package.xml in tutorial_interfaces.
    ```txt 
    find_package(geometry_msgs REQUIRED)
    find_package(rosidl_default_generators REQUIRED)
      rosidl_generate_interfaces(${PROJECT_NAME}
      "msg/Num.msg"
      "msg/Sphere.msg"
      "srv/AddThreeInts.srv"
      DEPENDENCIES geometry_msgs # Add packages that above messages depend on, in this case geometry_msgs for Sphere.msg
    )
    ```
    in this case, tutorial_interfaces package should be changed.
    ```
    <depend>geometry_msgs</depend>
    <buildtool_depend>rosidl_default_generators</buildtool_depend>
    <exec_depend>rosidl_default_runtime</exec_depend>
    <member_of_group>rosidl_interface_packages</member_of_group>
    ```
3. build and source tutorial_interfaces. Do via the above project's way. But this whole project is just like a interface used for reference.Test it using command:

    ```shell
    ros2 interface show tutorial_interfaces/msg/Num
    ros2 interface show tutorial_interfaces/msg/Sphere
    ros2 interface show tutorial_interfaces/srv/AddThreeInts
    ```

4. Chages to pub/sub model. WHAT will be changed in xml and cmakelist:
    + Include the header:
    ```cpp
    ros2 interface show tutorial_interfaces/msg/Sphere
    ```
    + Get the message sturct in the cpp:
    ```
    auto message = tutorial_interfaces::msg::Num();  // including the class name 
    ```
    + Change the CMakeList.txt
    ```cmake
    #...
    find_package(ament_cmake REQUIRED)
    find_package(rclcpp REQUIRED)
    find_package(tutorial_interfaces REQUIRED)                      # CHANGE

    add_executable(talker src/publisher_member_function.cpp)
    ament_target_dependencies(talker rclcpp tutorial_interfaces)    # CHANGE

    add_executable(listener src/subscriber_member_function.cpp)
    ament_target_dependencies(listener rclcpp tutorial_interfaces)  # CHANGE

    install(TARGETS
      talker
      listener
      DESTINATION lib/${PROJECT_NAME})

    ament_package()
    ```
    REMEMBER adding the interface in the xml file of the packages in the msg/srv model.
    ```xml
    <depend>tutorial_interfaces</depend>
    ```
5. Build and run.
   > colcon build --packages-select cpp_pubsub
   > ros2 run cpp_pubsub talker
   > ros2 run cpp_pubsub listener

### pub/sub idl model

What are the differences in idl model?
***The structure of the ros_dds project***
![project structure](/commons/images/3141911-20230420160104997-895049304.png)

Acker.idl
```idl
#include "ros_dds/msg/timer.idl"
module ros_dds{
 module msg{
  struct Acker {
    ros_dds::msg::Timer stamp;
    float steering_tire_angle;
    float steering_tire_rotation_rate;
  };
 };
};
```
Timer.idl
```idl
module ros_dds {
  module msg {
    struct Timer {
      int32 sec;
      uint32 nanosec;
    };
  };
};
```
1. Just like the tutorial_interfaces interface before. Change the xml and the CMakeList file.
    ``` cmake
    cmake_minimum_required(VERSION 3.8)
    project(ros_dds)

    if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
      add_compile_options(-Wall -Wextra -Wpedantic)
    endif()

    # find dependencies
    find_package(ament_cmake REQUIRED)
    # uncomment the following section in order to fill in
    # further dependencies manually.
    # find_package(<dependency> REQUIRED)

    find_package(rosidl_default_generators REQUIRED)

    rosidl_generate_interfaces(${PROJECT_NAME}
    "msg/acker.idl"
    "msg/timer.idl" )


    if(BUILD_TESTING)
      find_package(ament_lint_auto REQUIRED)
      # the following line skips the linter which checks for copyrights
      # comment the line when a copyright and license is added to all source files
      set(ament_cmake_copyright_FOUND TRUE)
      # the following line skips cpplint (only works in a git repo)
      # comment the line when this package is in a git repo and when
      # a copyright and license is added to all source files
      set(ament_cmake_cpplint_FOUND TRUE)
      ament_lint_auto_find_test_dependencies()
    endif()

    ament_package()
    ```
    version and description maintainer should be changed in a engineering project
    ```xml
    <?xml version="1.0"?>
    <?xml-model href="http://download.ros.org/schema/package_format3.xsd" schematypens="http://www.w3.org/2001/XMLSchema"?>
    <package format="3">
      <name>ros_dds</name>
      <version>0.0.0</version>
      <description>TODO: Package description</description>
      <maintainer email="quintin@todo.todo">quintin</maintainer>
      <license>TODO: License declaration</license>

      <buildtool_depend>ament_cmake</buildtool_depend>

      <test_depend>ament_lint_auto</test_depend>
      <test_depend>ament_lint_common</test_depend>


      <build_depend>rosidl_default_generators</build_depend>
      <exec_depend>rosidl_default_runtime</exec_depend>
      <member_of_group>rosidl_interface_packages</member_of_group>


      <export>
        <build_type>ament_cmake</build_type>
      </export>
    </package>
    ```
    Here I met a interesting question, when I named the `timer.idl` internal struct as `struct Time`, different with the filename, a error raised.
    ![Idl and internal struct should be used in the same name](/commons/images/3141911-20230419165116868-2046153484.png)
    So, should the .idl filename be consistant with the internal struct name?
    Here is the answer by NewBing.
    ![NewBing's answer](/commons/images/3141911-20230419172302295-875512889.png)
    And it seems not clearly clarified in the ROS2 Humble document.
    Finally, that is because of the parser in the rosidl_parser
    ![The reason is defined in rosidl_parser](/commons/images/3141911-20230419171924791-2047891303.png)
2. Add idl interfaces `ros_dds` in the ros_pub/sub.
![ros_dds interfaces](/commons/images/3141911-20230530154444457-87839594.png)
3. Changes in the `CMakelists.txt`:
    ```cmake 
    cmake_minimum_required(VERSION 3.8)
    project(apros)

    if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
      add_compile_options(-Wall -Wextra -Wpedantic)
    endif()

    # find dependencies
    find_package(ament_cmake REQUIRED)
    # uncomment the following section in order to fill in
    # further dependencies manually.
    # find_package(<dependency> REQUIRED)
    <depend>rclcpp</depend>
    find_package(rclcpp REQUIRED)
    # find_package(std_msgs REQUIRED)
    find_package(apidl_interfaces REQUIRED)   # CHANGE

    add_executable(talker src/publisher_apidl.cpp)
    # ament_target_dependencies(talker rclcpp std_msgs)
    ament_target_dependencies(talker rclcpp apidl_interfaces)    # CHANGE
    add_executable(listener src/subscriber_apidl.cpp)
    # ament_target_dependencies(listener rclcpp std_msgs)
    ament_target_dependencies(listener rclcpp apidl_interfaces)    # CHANGE

    if(BUILD_TESTING)
      find_package(ament_lint_auto REQUIRED)
      # the following line skips the linter which checks for copyrights
      # comment the line when a copyright and license is added to all source files
      set(ament_cmake_copyright_FOUND TRUE)
      # the following line skips cpplint (only works in a git repo)
      # comment the line when this package is in a git repo and when
      # a copyright and license is added to all source files
      set(ament_cmake_cpplint_FOUND TRUE)
      ament_lint_auto_find_test_dependencies()
    endif()


    install(TARGETS
    talker
    listener
    DESTINATION lib/${PROJECT_NAME})

    ament_package()

    ```
4. Changes in the cpp file. Here because a new project named `idl_pubsub` is created, so the interface generated using the ros_dds should be added into the header in the pub/sub cpp, the hpp is generated by `colcon` command in the ROSWORKSPACE's build folder.
    ```cpp
    #include "ros_dds/msg/acker.hpp"      // CHANGE
    ```
5. Build and run:
    ![idl ros pubsub model ](/commons/images/3141911-20230419181145170-1958054827.png)

## FastDDS prep
In this step, first learn how to generate a project using `.idl`, following the [tutorial](https://fast-dds.docs.eprosima.com/en/latest/fastdds/getting_started/simple_app/simple_app.html) in the FastDDS website.
![FastDDS project structure](/commons/images/3141911-20230420162752830-710419901.png)

When using the templates from the fastdds website, there are several places need to change.
To be consistant with the idl file in ROS configuration, need to add the outer msg and moudle name.

In the tutorial file, the idl struct is like this:

```idl
struct HelloWorld
{
   unsigned long index;
   string message;
}
```
And its mapped classes in the cxx file just like this, where HelloWorldPubSubType() Defined by fastddsgen?
![fastddsgen from original Hello world idl file ](/commons/images/3141911-20230420165530116-141696030.png)

See what's different with the ROS2 generated? 
ROS2's idl generated file using its own idl:
![Using ROS2's idl standard to generate file](/commons/images/3141911-20230420170032352-1764454826.png)

Using ROS2's idl standard to generate file, because they should follow the ==[standard](https://docs.ros.org/en/humble/Concepts/About-ROS-Interfaces.html)== ruled by OMG 4.2, and in a `module{msg{...}}` form, so there are several inconsistencies between the fastdds idl and ros2 idl, change fastdds idl to a ROS2 form, or when they are communicating with each other, their struct will come from a different namespace, for example, fastdds: `acker` and the ROS2: `ros_dds::msg::acker`, so just place the message struct in the same layer.

<font face="TIMES" size=5>**Publisher and Subscriber in FastDDS method**</font>

1. Change the structure of the idl file, and regen from the idl file.
2. Pay attention to the topic's creation process.
    How the topic is created? 
    ==[TopicQos controls the behavior of the Topic. Internally it contains the following QosPolicy objects](https://fast-dds.docs.eprosima.com/en/latest/fastdds/dds_layer/topic/topic/topic.html?highlight=create_topic#default-topicqos)==
    In the type_ registration process:
    ```cpp
    // RosDDSildPublisher.cpp, modified from HelloWorldPublisher.cpp
      RosDDSidlPublisher()
      : participant_(nullptr)
      , publisher_(nullptr)
      , topic_(nullptr)
      , writer_(nullptr)
      , type_(new ros_dds::msg::AckerPubSubType())
    {
    }
    ```
    Check the formation in the idl gernerated Type.h and Type.cxx
    print the type name of `AckerPubSubType`
    ![print topic name for publisher in FastDDS](/commons/images/3141911-20230420123143474-650564601.png)
    ```cpp
    // ackerPubSubType.h, generated from idl file by fastddsgen
    namespace ros_dds {
    namespace msg {
        AckerPubSubType::AckerPubSubType()
        {
            // setName("ros_dds::msg::Acker");
            //! Modify here
            setName("ros_dds::msg::dds_::Acker_");
            auto type_size = Acker::getMaxCdrSerializedSize();
            type_size += eprosima::fastcdr::Cdr::alignment(type_size, 4); /* possible submessage alignment */
            m_typeSize = static_cast<uint32_t>(type_size) + 4; /*encapsulation*/
            m_isGetKeyDefined = Acker::isKeyDefined();
            size_t keyLength = Acker::getKeyMaxCdrSerializedSize() > 16 ?
                    Acker::getKeyMaxCdrSerializedSize() : 16;
            m_keyBuffer = reinterpret_cast<unsigned char*>(malloc(keyLength));
            memset(m_keyBuffer, 0, keyLength);
        }
    ```
    ```cpp
    // Create the publications Topic
    // topic_ = participant_->create_topic("rt/rosidltopic", // [Why rt?]
    topic_ = participant_->create_topic("rosidltopic",
            //  "ros_dds::msg::dds_::Acker_", 
            type_.get_type_name(), // at the setName in ackerPubSubType.h
              TOPIC_QOS_DEFAULT); 
    ```
    Some errors due to mistakes in RosDDSidlPublisher.cpp: 
    ![fastdds configuration errors](/commons/images/3141911-20230419203626354-1198864300.png)
3. Modify Subscriber.cpp in a same way.
    ![Subscriber topic wrongly setted](/commons/images/3141911-20230420102843272-52799788.png)
4. build and run
Successfully build and communicate in FastDDS
![build and communicate with the same topic](/commons/images/3141911-20230420103400376-14993912.png)

# Communication betweent ROS2 and FastDDS
This part will divided into three part:
1. FastDDS Publisher and ROS2 Subscriber
2. ROS2 Publisher and FastDDS Subscriber
3. What's different?

ROS2's pub/sub model uses Domain ID and a specific topic, how to figure them out when communicate?
+ In ROS2, the primary mechanism for having different logical networks share a physical network is known as the Domain ID. ROS 2 nodes on the same domain can freely discover and send messages to each other, while ROS 2 nodes on different domains cannot. ==[All ROS 2 nodes use domain ID 0 by default](https://docs.ros.org/en/foxy/Concepts/About-Domain-ID.html)==.
+ To configure a publisher in ROS2 using FastDDS, you can define a <data_writer> profile with attribute profile_name=topic_name, where topic_name is the name of the topic prepended by the node namespace (which defaults to ""? if not specified), i.e. the node's namespace followed by topic name used to create the publisher.
    Change the default domain by using below command:
    ```shell
    export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
    export ROS_DOMAIN_ID= [IDYOUWANT]
    ```
    Change the default topic can modify the code or from command or from xml.
    ```shell 
    export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
    export FASTRTPS_DEFAULT_PROFILES_FILE=path/to/xml/ros_example.xml
    export RMW_FASTRTPS_USE_QOS_FROM_XML=1
    ros2 run demo_nodes_cpp talker
    ``` 
    let’s go to the first terminal, stop the current publisher by pressing CTRL+C, and let’s relaunch the publisher but now setting the ROS_DOMAIN_ID=1.
    ```shell
    ROS_DOMAIN_ID=1 ros2 topic pub -r 1 /string_topic std_msgs/String "{data: \"Hello from my 2ND domain\"}"
    ```
    ==[How to look up the topics what ROS2 is using?](https://www.theconstructsim.com/separating-ros2-environments-ros_domain_id-ros2-concepts-in-practice/)==
    ==[OR HERE IN OFFICIAL](https://integration-service.docs.eprosima.com/en/latest/examples/same_protocol/ros2_change_domain.html)==

    The topics can be created both Publisher or Subscriber.
    ```shell
    ROS_DOMAIN_ID=1 ros2 topic list
    /parameter_events
    /rosout
    /string_topic
    # OR
    ros2 topic list
    /parameter_events
    /string_topic
    ``` 
    ![checking topic name during running publisher and listener](/commons/images/3141911-20230420111150181-1351553594.png)

## FastDDS Publisher and ROS2 Subscriber

To have a ROS2 subscriber communicate with a FastDDS publisher, you need to make sure that both the subscriber and publisher are using the same Domain ID and topic name. You can follow these steps to set up communication between a ROS2 subscriber and a FastDDS publisher:

1. Set up the FastDDS environment by defining the data type of the messages that will be sent by the publisher and received by the subscriber using Fast DDS-Gen application1.

2. Create a FastDDS publisher with the desired Domain ID and topic name.

3. Set up the ROS2 environment by creating a package and writing a subscriber node.

4. Make sure that the ROS2 subscriber is using the same Domain ID and topic name as the FastDDS publisher.

5. Run both the FastDDS publisher and ROS2 subscriber.
<font face="TIMES" size=5>**Topic name and Type name change**</font>

+ ==[Here](https://github.com/eProsima/Fast-DDS/issues/1995)== is talking a problem about chang the `topic` in fastdds cpp to `rt/topic`.
When communicating between ROS2 and FastDDS, it is necessary to add the rt/ prefix to the topic name. This is because ROS2 uses a different naming convention for topics than FastDDS, and adding the rt/ prefix allows FastDDS to recognize the topic as a ROS2 topic.
The ability for a ROS2 node to talk with a raw fastrtps node (without modification) has a few caveats. The first is going to be getting message type names to match up. The second is around topic names. ==[By default, ros2 nodes prefix all topic names with rt/, rq/, or rr/ (topic, request, response respectively...)](https://answers.ros.org/question/380642/ros2-to-fastrtps-interoperability/)== There is a avoid_ros_namespace_conventions parameter in the rclcpp::QoS structure, but even with that enabled, it forces every topic you make to have a leading / in the topic. Bottom line: if your raw fastrtps topics start with /, then you're fine. Otherwise you'll need to change your raw fastrtps nodes to have that / prefix the topic names.
+ Here is talking a problem about setting the type name into a specific form.
    Why? because when generate files from idl, not use the `--typeros2` option
    ```cpp
    // call get_type_name(), and its return a string, which is set at the XXXPubSubType.cxx 
    // original
    topic_ = participant_->create_topic("rt/rosidltopic",
            //  "ros_dds::msg::dds_::Acker_", 
            type_.get_type_name(),
              TOPIC_QOS_DEFAULT);
    // ackerPubSubTypes.cxx    
    #include <fastcdr/FastBuffer.h>
    #include <fastcdr/Cdr.h>

    #include "ackerPubSubTypes.h"

    using SerializedPayload_t = eprosima::fastrtps::rtps::SerializedPayload_t;
    using InstanceHandle_t = eprosima::fastrtps::rtps::InstanceHandle_t;

    namespace ros_dds {
        namespace msg {
            AckerPubSubType::AckerPubSubType()
            {
                // setName("ros_dds::msg::Acker");
                setName("ros_dds::msg::dds_::Acker_");
                auto type_size = Acker::getMaxCdrSerializedSize();
                type_size += eprosima::fastcdr::Cdr::alignment(type_size, 4); /* possible submessage alignment */
                m_typeSize = static_cast<uint32_t>(type_size) + 4; /*encapsulation*/
                m_isGetKeyDefined = Acker::isKeyDefined();
                size_t keyLength = Acker::getKeyMaxCdrSerializedSize() > 16 ?
                        Acker::getKeyMaxCdrSerializedSize() : 16;
                m_keyBuffer = reinterpret_cast<unsigned char*>(malloc(keyLength));
                memset(m_keyBuffer, 0, keyLength);
            } // ...
        } // ...
    }//// ...
    ```
if ROS2 exists any of Talker(Publisher) or Listener(Subscriber), there will exist different topic, which can be displayed by the command `ros2 topic list`.
And different topics that being used by ROS2 will be listed.
Success! FastDDS Publisher with ROS Subscriber
![Success! FastDDS Publisher with ROS Subscriber](/commons/images/3141911-20230420132306505-360312976.png)
## ROS2 Publisher and FastDDS Subscriber
Just the same process as former. 
Success! ROS Publisher with FastDDS Subscriber
![Success! ROS Publisher with FastDDS Subscriber](/commons/images/3141911-20230420134136098-1976802104.png)

## What's different?

+ idl file generation
+ About the change of the topic.
+ Where to change the typename, or there is no possible to communication between FastDDS and ROS2.
![change in the ackerPubSubTypes](/commons/images/3141911-20230420133245516-87857339.png)

# What will be different if use fastddsgen generate the idl with `--typeros2` option?
So what will be different if generate files from `--typeros2` option?
It will help you Set the Type Name properly, That's all.
![The differences](/commons/images/3141911-20230420214257579-64530029.png)

![fastddsgen use option --typeros2](/commons/images/3141911-20230420213641415-1387772611.png)
