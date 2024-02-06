---
title: AutoSAR-AP gen code review
date: 2023-10-02
categories: [AutoSAR]
tags: [autosar]     # TAG names should always be lowercase
published: false
---

<center><font face='TIMES' size='6'> AP-GEN code reveiw</font></center>
[TOC]
# AP_GEN Project Overview
AP_GEN is a project that uses ara-api and database config version of `arxml` to generate codes from different network binding (fastdds available now only).
```text
Linearx/
├── 3rd-party
│   └── eProsima
│       ├── Fast-CDR
│       ├── Fast-DDS
│       └── foonathan_memory_vendor
├── apgen
│   ├── ap_build.py
│   ├── ap_gen.py
│   ├── ara_gen
│   │   ├── ara_genparser.py
│   │   ├── doc
│   │   ├── manifest
│   │   └── script
│   ├── cmake_template
│   │   ├── cmakelist_dynamic.txt.in
│   │   ├── CMakeLists_exist_app.j2
│   │   ├── CMakeLists_exist_exec.j2
│   │   ├── CMakeLists.txt.j2
│   │   ├── main_xxx.cpp
│   │   ├── main_xxx_shcedule_client.cpp
│   │   ├── xxx_activity_proxy.cpp
│   │   ├── xxx_activity_proxy.h
│   │   ├── xxx_activity_skeleton.cpp
│   │   └── xxx_activity_skeleton.h
│   ├── fastddsgen
│   │   ├── scripts
│   │   └── share
│   └── utils
│       ├── apgen_args.py
│       ├── apgen_template_renderer.py
│       └── __pycache__
├── applications
│   ├── exec
│   │   ├── CMakeLists.txt
│   │   ├── files
│   │   ├── include
│   │   ├── INSTALL.md
│   │   ├── LICENSE
│   │   ├── README.md
│   │   └── src
│   ├── readme.txt
│   └── StateManager
│       ├── calibration
│       ├── CMakeLists.txt
│       ├── include
│       └── src
├── genRuntime.py
├── include
│   ├── apd
│   │   ├── crc
│   │   ├── manifestreader
│   │   ├── platform
│   │   ├── rest
│   │   ├── rng
│   │   └── test
│   ├── ara
│   │   ├── com
│   │   ├── core
│   │   ├── exec
│   │   ├── log
│   │   └── per
│   ├── ara_core.cxx
│   ├── ara_core.h
│   ├── ara_corePubSubTypes.cxx
│   ├── ara_corePubSubTypes.h
│   ├── dds_base_types.cxx
│   ├── dds_base_types.h
│   ├── dds_idl
│   │   ├── ara_core.idl
│   │   ├── dds_base_types.idl
│   │   ├── dds_rpc.idl
│   │   └── type_converter_ara_core.h
│   ├── dds_rpc.cxx
│   ├── dds_rpc.h
│   ├── dds_rpcPubSubTypes.cxx
│   ├── dds_rpcPubSubTypes.h
│   ├── json
│   │   ├── allocator.h
│   │   ├── assertions.h
│   │   ├── config.h
│   │   ├── forwards.h
│   │   ├── json_features.h
│   │   ├── json.h
│   │   ├── reader.h
│   │   ├── value.h
│   │   ├── version.h
│   │   └── writer.h
│   ├── libipc
│   │   ├── buffer.h
│   │   ├── condition.h
│   │   ├── def.h
│   │   ├── export.h
│   │   ├── ipc.h
│   │   ├── mutex.h
│   │   ├── pool_alloc.h
│   │   ├── rw_lock.h
│   │   ├── semaphore.h
│   │   └── shm.h
│   ├── scheduler
│   │   ├── app
│   │   ├── caros_channel
│   │   ├── caros_sched
│   │   ├── caros_sched_client
│   │   └── caros_utility
│   └── wrsomeip
│       ├── appSrvList.h
│       ├── eventGroupEntryType.h
│       ├── localClientEndpoint.h
│       ├── localServerEndpoint.h
│       ├── osal.h
│       ├── routerManager.h
│       ├── serviceApplication.h
│       ├── serviceDiscovery.h
│       ├── serviceEntryType.h
│       ├── serviceList.h
│       ├── someipConfiguration.h
│       ├── someipLogger.h
│       ├── someipMessage.h
│       ├── someipPayload.h
│       ├── someipSdTypes.h
│       ├── someipSerial.h
│       ├── someipService.h
│       ├── someipTimer.h
│       ├── someipTypes.h
│       ├── someipUtil.h
│       ├── tcpClientEndpoint.h
│       ├── tcpServerEndpoint.h
│       ├── udpClientEndpoint.h
│       ├── udpServerEndpoint.h
│       └── wrsomeip.h
├── lib
│   ├── cmake
│   │   ├── apd-applications-arxmls
│   │   ├── apd-common-machine-arxmls
│   │   ├── apd-crc
│   │   ├── apd-interfaces-arxmls
│   │   ├── apd-manifestreader
│   │   ├── apd-network-arxmls
│   │   ├── ApdPlatform
│   │   ├── apd-radarfusionddsmachine-arxmls
│   │   ├── apd-rest
│   │   ├── apd-rng
│   │   ├── apd-testutils
│   │   ├── apd-wrsomeip
│   │   ├── ara-arxmls
│   │   ├── ara-com
│   │   ├── ara-core
│   │   ├── ara-core-types
│   │   ├── ara-deterministic-scheduler
│   │   ├── ara-deterministic-scheduler-client
│   │   ├── ara-exec-deterministic-client
│   │   ├── ara-exec-execution-client
│   │   ├── ara-exec-find-process-client
│   │   ├── ara-exec-state-client
│   │   ├── ara-gen
│   │   ├── ara-log
│   │   ├── ara-per
│   │   ├── e2e
│   │   └── e2exf
│   ├── libapd_crc.a
│   ├── libapd_manifestreader.a
│   ├── libapd_platform.a
│   ├── libapd_rest.a
│   ├── libapd_rng.a
│   ├── libara_com.a
│   ├── libara_core.a
│   ├── libara_core_types.a
│   ├── libara_deterministic_scheduler.a
│   ├── libara_deterministic_scheduler_client.a
│   ├── libara_exec_deterministic_client.a
│   ├── libara_exec_execution_client.a
│   ├── libara_exec_find_process_client.a
│   ├── libara_exec_state_client.a
│   ├── libara_fastddsbinding.a
│   ├── libara_fileaccess.a
│   ├── libara_keyvaluestorage.a
│   ├── libara_kvsparser.a
│   ├── libara_kvstype.a
│   ├── libara_log.a
│   ├── libara_manifestaccess.a
│   ├── libara_per_utility.a
│   ├── libara_redundancy.a
│   ├── libara_updateper.a
│   ├── libara_vsomeipbinding.a
│   ├── libara_wrsomeipbinding.a
│   ├── libdlt.so
│   ├── libdlt.so.2
│   ├── libdlt.so.2.18.8
│   ├── libe2e.so -> libe2e.so.4.2.2
│   ├── libe2e.so.4.2.2
│   ├── libe2exf.so -> libe2exf.so.4.2.2
│   ├── libe2exf.so.4.2.2
│   ├── libfastcdr.so.1
│   ├── libfastrtps.so.2.8
│   ├── libipc.a
│   ├── libjsoncpp.a
│   ├── libwrsomeip.so -> libwrsomeip.so.1
│   ├── libwrsomeip.so.1 -> libwrsomeip.so.1.0.0
│   ├── libwrsomeip.so.1.0.0
│   └── pkgconfig
│       ├── apd-crc.pc
│       ├── apd-manifestreader.pc
│       ├── apd_platform.pc
│       ├── apd-rest.pc
│       ├── apd-testutils.pc
│       ├── ara-core.pc
│       ├── ara-core-types.pc
│       ├── ara-deterministic-scheduler-client.pc
│       ├── ara-deterministic-scheduler.pc
│       ├── ara-exec-deterministic-client.pc
│       ├── ara-exec-execution-client.pc
│       ├── ara-exec-find-process-client.pc
│       ├── ara-exec-state-client.pc
│       ├── ara-log.pc
│       ├── libe2e.pc
│       └── libe2exf.pc
├── Readme
├── share
│   └── PH3B.dat
└── src
    └── RadarFusionMachine
        ├── fusion
        ├── machines
        └── radar
```

1. Entry python file
`Linearx/apgen/ap_gen.py`
2. Entry code
`main()` function in `ap_gen.py`
![entry code for ap_gen](/commons/images/3141911-20230509110508388-882737435.png)
3. ModelGenerate() function in main()
![ModelGenerate function](/commons/images/3141911-20230509110832301-855265920.png)
4. get config in the ModelGenerate() class.
data from `.dat` file in `Linearx/share`.
In the render() function, the compeleted model will be generated in the generateApModel(), we can know from code that mulit-processes are working on one machines and there is a specificial function `RunAraGengenargs()`
![GenerateApModel](/commons/images/3141911-20230509122153177-135181853.png)
    ```python
        gen_args = [self.ara_gen, f"--dburl={dburl}", "--dds-libs=fastdds",\
            f"--software-components={component.fqn}",f"--processes={process.fqn}",\
            f"--output={process_output}"]
        RunAraGengenargs(gen_args)
    ```
    ![RunAraGengenargs function](/commons/images/3141911-20230509134754928-1233745742.png)

    RunAraGengenargs function is called in three places, and in each time, args are prepared for it before it be called. ModuleGenerator is called inside `RunAraGengenargs(argv)` function. And what is special is the function contains render() function, and this `render()` is belongs to `ModuleGenerator` class, which is different with the `render()` function in `main()` that is a member of `ModelGenerate` class. And through `Generator` class in `Linearx/apgen/ara_gen/script/generator/generator/generator.py`, a whole model is returned.
    ![Module Generator](/commons/images/3141911-20230509135715974-1437600994.png)

# Generator() function in ModuleGenerator Class

This is the `GeneratorSettings` function which is called in init function in `Generator` class.

The getattr () function returns the value of the specified attribute from the specified object. Required. An object. Optional. The value to return if the attribute does not exist. `getattr(object, name[, default])`. default -- The default return value, if this parameter is not supplied, AttributeError will be triggered when there is no corresponding property.

![GeneratorSetting function in render](/commons/images/3141911-20230509143652774-2065479191.png)
```python

    # Different Libs, now only dds is need
    _LIB_BINDING = {
        'someip' : {
            'vsomeip' : {
                'module' : _LIB_BINDING_PACKAGE + '.vsomeip_binding',
                'class' : 'VsomeipBinding',
            },
            'wrsomeip' : {
                'module' : _LIB_BINDING_PACKAGE + '.wrsomeip_binding',
                'class' : 'WRsomeipBinding',
            }
        },
        'user_defined' : {
            'vsomeip' : {
                'module' : _LIB_BINDING_PACKAGE + '.vsomeip_binding',
                'class' : 'VsomeipBinding',
            }
        },
        'dds' : {
            'opendds' : {
                'module' : _LIB_BINDING_PACKAGE + '.opendds_binding',
                'class' : 'OpenDdsBinding',
            },
            'fastdds' : {
                'module' : _LIB_BINDING_PACKAGE + '.fastdds_binding',
                'class' : 'FastDdsBinding',
            }
        }
    }
    # Lib binding DDS 
    self._dds_bindings = []
    for name in self._settings.dds_libs:
        module = importlib.import_module(Generator._LIB_BINDING['dds'][name]['module'])
        Binding = getattr(module, Generator._LIB_BINDING['dds'][name]['class'])
        binding = Binding(self._renderer, self.net_bindings_root_dir, self.machines_root_dir)
        self._dds_bindings.append(binding)
```

And `fastdds_binding.py` at `Linearx/apgen/ara_gen/script/generator/generator/lib_binding/fastdds_binding.py` is a template generator file used to generate different `service`, `proxy`, `machine`, `method` .etc from jinjia2 template.
![fastdds bindings](/commons/images/3141911-20230509152258886-486175739.png)

Examp. `generate_proxy(self, service: ServiceView)` function.
![generate_proxy](/commons/images/3141911-20230509152842633-1875375339.png)
`xxx.h.j2` file in fastdds_binding directory `Linearx/apgen/ara_gen/script/generator/templates/fastdds_binding`
![jinjia2 template](/commons/images/3141911-20230509154116128-2029389670.png)

member function for Generator: services in different bindings:

```python 
def _generate_required_service_bindings(self, bindings, services):
    for binding in bindings:
        for service in services:
            binding.generate_service_desc(service)
            binding.generate_proxy(service)
```

# DBParser in render() function in MoudleGenerator Class

![DBParser in render()](/commons/images/3141911-20230509164447490-482857800.png)
First, check the db file in the share directory: `Linearx/share/PH3B.dat`
View the structure of the database in sqlitebrowser
![PH3B database in sqlitebrowser](/commons/images/3141911-20230509163238759-152378951.png)

> select * from AP_MACHINES;

![Check AP_machines](/commons/images/3141911-20230511175418869-2064422796.png)

> select * from AP_ALEMENT;

![Check AP_ELEMENT](/commons/images/3141911-20230511181930125-817646423.png)

> select * from AP_PROCESS;

![views from AP_PROCESS](/commons/images/3141911-20230511193642515-237139527.png)

`sqlexec()` function in DBParser(), which contains all kinds of different operation to sqlite3, including `connectToDb`, `disconnectToDb`, `attachDB`, `checkTableCotentShortNameAvailable`, `dropTable` .etc.
![Init funciton of DBParser](/commons/images/3141911-20230509164630887-1513130875.png)



## InterfaceBuilder in DBParser

![InterfaceBuilder in DBParser](/commons/images/3141911-20230509165543688-234890329.png)

Parser includes different contents connect to analyse database file and parser database file and convert it to configuration, which is the same as what of `.arxml` file. `Linearx/apgen/ara_gen/script/generator/parser`
![Parser directory](/commons/images/3141911-20230509165937439-1672609263.png)

## NetworkBindingBuilder
DDSserviceDeployment in NetworkBindingBuilder, and `service_id`, `instance_id`, `domian_id`, `qos_profile` are all in this function.
![NetworkBindingBuilder](/commons/images/3141911-20230509170641061-1985719782.png)

# Generator.generate() function which is called in ModuleGenerator.render()

## the create of the intermediate in DBParser

intermediate_model is created in the DBParser.create_intermediate_model():
![create_intermediate_model](/commons/images/3141911-20230510105405645-1605642319.png)

```python
    return {
        "processes": self._get_processes_views(processes),
        "machines": self._get_machines_views(machines),
        "components": self._get_components_views(swc),
        "errors": self._interface_builder.get_error_domain_views()
    }
```
Take the function `_get_process_views` as the example 
![_get_process_views](/commons/images/3141911-20230510105753099-1175457055.png)
Class `ProcessView`
![class processview](/commons/images/3141911-20230510110128079-273777021.png)

The same as class `MachineView`:
![class MachineView](/commons/images/3141911-20230510111358951-1934103681.png)

The Base Class `View` is for any View object, pay attention to the attribute `__getattr__`.
![Base View class for any model object](/commons/images/3141911-20230510110930402-1309145660.png)

## the generation of the intermdeiate_model in Generator
generate the intermediate_model in Class `Generator`.
![generate the intermediate_model ](/commons/images/3141911-20230510104620409-1316610529.png)

Three different targets that need generating.

![generate function in Generator](/commons/images/3141911-20230509172348794-1259191834.png)

The generation of the `model['machines']` in the Generator.generator:
The generation process include:

+ _generate_network_binding_config()
+ _generate_machine_manifest()
+ _generate_network_config()
+ _generate_dlt_config()

![_generate_machines in Generator](/commons/images/3141911-20230510112122439-1931587201.png)

# Back to ModelGenerate.generateApModel()

![ModelGenerate.generateApModel()](/commons/images/3141911-20230510113402535-1036949053.png)

## generateIdl(process_output,common_idl_dir)

![generateIdl](/commons/images/3141911-20230510135431095-2025004274.png)

**Modifications of the `generateIdl`**

![Modifications of the `generateIdl`](/commons/images/3141911-20230510135215683-1418739736.png)

Why modificaiton？

The previous generation process was generated one by one, and it was generated at one time after modification.

## generatecommonCmake(process_output)

![generatecommonCmake](/commons/images/3141911-20230510113820000-1239911047.png)

The whole process is like to load a dynamic template and set all the command in the output cmakeList file.

![cmake_dynamic.txt.in](/commons/images/3141911-20230510141012874-820147609.png)

## generateUserPart(process, instance_specifier, component, process_output)

![generateUserPart](/commons/images/3141911-20230510133612671-558239452.png)

The processes to generate `skeleton_h_cpp` and `proxy_h_cpp` are the same as the process to generate `main_cpp`, which are all generation processes from `template`, final results are written as `.h`,`.cpp`,`.hpp` in the `src` directory.

![generateUserCode](/commons/images/3141911-20230510133842961-686647422.png)

# The Generation Result

==<font face='TIMES' size='5'>Overview of the `src` directory.</font>==

```txt
.
└── RadarFusionMachine
    ├── fusion
    │   ├── CMakeLists.txt
    │   ├── fusion
    │   │   ├── build
    │   │   ├── CMakeLists.txt
    │   │   ├── include
    │   │   └── src
    │   ├── includes
    │   │   ├── ara
    │   │   └── impl_type_bytearray.h
    │   ├── net-bindings
    │   │   ├── ara_com_main-fusion.cpp
    │   │   └── fastdds
    │   └── processes
    │       ├── fusion_MANIFEST.json
    │       ├── fusion_phm.json
    │       └── fusion_SI_MANIFEST.json
    ├── machines
    │   ├── fastdds
    │   │   └── RadarFusionMachine_rtps.ini
    │   ├── RadarFusionMachine_dlt.conf
    │   ├── RadarFusionMachine_etc_network_interfaces
    │   └── RadarFusionMachine_machine_manifest.json
    └── radar
        ├── CMakeLists.txt
        ├── includes
        │   ├── ara
        │   └── impl_type_bytearray.h
        ├── net-bindings
        │   ├── ara_com_main-radar.cpp
        │   └── fastdds
        ├── processes
        │   ├── radar_MANIFEST.json
        │   ├── radar_phm.json
        │   └── radar_SI_MANIFEST.json
        └── radar
            ├── build
            ├── CMakeLists.txt
            ├── include
            └── src

23 directories, 18 files
```
## RadaFusionMachine compling

1. OpenSSL used in the project is too old, version 1.1.1

openssl is the old version, 1.1.1 has been expired at 7th Sept.
openssl3 is available now. 
comple fusion failed:
![complie fusion ](/commons/images/3141911-20230510180138148-1390184682.png)

Reason:
![Reason for compling failure](/commons/images/3141911-20230510183513796-984149036.png)

Method:
Download the source code from `openssl` website to make and install.

2. error while loading shared libraries: libe2e.so.4.2.2: cannot open shared object file: No such file or directory
![error while loading shared libraries](/commons/images/3141911-20230511105926547-471541720.png)
Reason:
In fact, in the cmakelist, lib has been included, but need to export as the environment variables.
![export error fix](/commons/images/3141911-20230511110209029-1580957232.png)

3. GCC version

![gcc version error](/commons/images/3141911-20230511124540044-1638707245.png)

Reason:
sleep_for and sleep_until are std conponents of  old version gcc.

Method:
Using update-alternatives to set different priority for using different versions of gcc. 
Recommend to use config
> sudo apdate-alternatives --config gcc
> sudo apdate-alternatives --config g++
and choose the version needed.
![gcc version](/commons/images/3141911-20230511125642532-1549826897.png)

Using ap_build.py to build, and export when necessary 
![ap_build to build the project](/commons/images/3141911-20230511150332931-1249017271.png)

run code at /opt/radar/
run code at /opt/fusion/

./bin/fusion
./bin/radar

![result](/commons/images/3141911-20230511161116143-1653270650.png)
