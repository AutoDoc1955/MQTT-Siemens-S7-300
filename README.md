# MQTT-Siemens-S7-300
MQTT library block written in SCL for S7-300 with *internal* (PN) or *external* (CP) Ethernet.

This started as a port of [knolleary's MQTT library](https://github.com/knolleary/pubsubclient) for Arduino & ESP8266.
The implementation is following the MQtt v3.1.1 protocol documentation (http://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html)

*Main functionallity complete, still work in Progress!*

# Purpose and Possibilities
MQTT is a popular communications protocol in the IoT space. Its demand on most types of networks and CPUs make it a good option for M2M applications. MQTT devices can easily be utilized to send a message anywhere in the world.

##### What does this mean for PLCs & Industrial Automation?
MQTT will enable a PLC to connect to the cloud without using proprietary hardware or protocols. PLC programmers can use it to build customized programs that send info to a web server, log plant data, and communicate with any other MQTT client device.
Developers can use it to build customized dashboards (physical, website, or mobile device), providing valuable data for analytics and business. MQTT enabled IO could serve as an inexpensive alternative for the PLC. (although not recommended for mission critical IO and for safety reasons)

### Current Test Scenario:
At this time, this code has been *successfully tested* on:

- CPU312C (312-5BF04-0AB0) with CP343-1 (343-1EX30-0XE0) external Ethernet
- CPU313C (313-5BG04-0AB0) with CP343-1 (343-1EX30-0XE0) external Ethernet
- CPU313C (6ES7313-5BF03-0AB0) with CP343-1 (343-1EX30-0XE0) external Ethernet (PLC with only 64KB RAM!)
- CPU314C-2 PN/DP (314-6EH04-0AB0) with CP343-1 (343-1EX30-0XE0) external Ethernet
- CPU315-2 PN/DP (315-2EH14-0AB0) with internal Ethernet


The PLC is connected to a Mosquitto MQtt broker.
All main functionallity has been testet: connect, disconnect, subscribe, unsubscribe, ping, publish.
I am locally connecting to a Mosquitto broker.

### Currently Limitations:

- not all MQtt policies described in the MQtt v3.1.1 standard are exactly standard conform implemented
  Especially the code must be reviewed wether it conforms to all the yellowish lines in the MQtt documentation
- subscribe and unsubscribe only for one topic at a time (you can subscribe multiple times if you need several topics)
- the Siemens PLC Ethernet adapters can send/receive *8192 bytes max.* per transmission. Please refer to the Simatic Manager help pages for the corresponding FBs/FCs
- Qos 2 handling for incoming publish messages not implemented

### Todo:
- state machine for tcp and mqtt should be harmonized
- MQtt policies review for the code

# Requirements
It should work with most S7-300/400 CPU's.
This code is written in Step7 SCL v5.3 SP1. It probably needs modification for it to compile in TIA.

# How to Compile/Use in your project

*Important:*
rbw: Import all files as "external sources" to the "sources" folder under the S7 Program in Step7.
There are two Makefiles: (rbw: Makefiles have suffix .inp.  You compile them using Siemens Step7 SCL editor.)

- Makefile-CP will compile the project with external ethernet support (CP, f.e. CP-343)
- Makefile-PN will compile the project with internal ethernet support (PN)

only compile the Makefile your PLC setup needs.

The following needs to be setup in you project in Simatic Manager:

1. In SCL Editor Options->Customize set "Create block numbers automatically"

2. You need to add the following Objects Library to your project:

   *Only needed for internal ethernet support (PN):*
   Library: Standard Library -> Communication Blocks -> Blocks
   - FB63  TSEND
   - FB64  TRCV
   - FB65  TCON
   - FB66  TDISCON
   

   *Only needed for external ethernet support (CP):*
   Library: SIMATIC_NET_PC -> CP300
   - FC5   AG_SEND
   - FC6   AG_RECV
   - FC10  AG_CTRL
   
   *Additional Objects needed:*
   Library: IEC Function Blocks
   - FC21  LEN
   - FC10  EQ_STRNG
   
 
   Library: System Function Block
   - SFB4  TON
   - SFC6  RD_SINFO
   - SFC20 BLKMOV
   - SFC58 WR_REC
   - SFC59 RD_REC

   
   *Important*: (rbw: if using CP logic) there will be an import conflict between FC10 AG_CTRL and FC10 EQ_STRING, as a solution rename one of the the FC-numbers during import
   
   
3. You have to compile the Makefile two times because there is still some DB creation problem.
   The second run will succeed. (rbw: My PN makefile compiled OK the first time.)

   
4. *Imporant:* You must call the MQTT function block in your OB1 program loop.
   
5. Optional: You may compile the MQTT_Example. (rbw: Getting syntax errors.  Not yet understood why.)
   
## Network Configuration
The MQTT FB can use the internal Ethernet adapter of a CPU (PN, choose MQTT_Main_PN.scl) or an external Ethernet adapter (CP, choose MQTT_Main_CP.scl).

Remarks for internal Ethernet (PN) adapter configuration:
- The IP-Address of the internal adapter has to be configured within the hardware configuration tool (HW Config), click on the PN-IO object
- The remote IP and Port parameters are configured via parameters for the MQTT Function Block (ipBlock1-4,ipPort)
- you must set the MQTT Functionblock parameter connectionID to a desired value, f.e. 1
Example for a MQTT FB call in OB1 configured for internal Ethernet (PN) usage:
rbw: NOTE: You must create MQTT instance block "DB71" or MQTT_Example won't compile.
*MQTT.DB71(net_config := DB_NET_CONFIG, connectionID := 1);*

 Remarks for external Ethernet (CP) adapter configuration:
 - The IP-Address of the internal adapter has to be configured within the hardware configuration tool (HW Config), click on the PN-IO object
 - The connection must be configured in Simatic Manager "Connections"
 - you must set the MQTT Functionblock parameter connectionID to the connection ID configured in Simatic Manager "Connections"
 - you must set the MQTT Functionblock parameter cpLADDR to the address of the CP module.
 Example for a MQTT FB call in OB1 configured for external Ethernet (CP)usage:
*MQTT.DB71(net_config := DB_NET_CONFIG, connectionID := 1, cpLADDR := W#16#100);*

## Setup memory footprint

### Setting up buffers
To reduce the memory usage of the MQtt DBs you can set the receive and transmit buffer sizes.

- Set tcpRecBuf array size in mqttData DB and TCP_RECVBUFFERSIZE in mqttGlobals to the same value (f.e. 8192) of your choice.
- Set TCP_MAXRECVSIZE in mqttGlobals DB to a value equal or smaller than TCP_RECVBUFFERSIZE
- Set tcpSendBuf array size in mqttData DB to a value of your choice
- Set buffer array size in mqttData DB to a value of your choice, but not smaller than the largest value of TCP_MAXRECVSIZE or tcpSendBuf, whoever is larger

Keep in mind: data from tcpRecBuf is transfered to buffer and also data from buffer is tranfered to tcpSendBuf within the Code. So match the sizes appropriately.

## Check connection status

You can check the connection status by monitoring two Flags in mqttData DB:

**mqttData.ethTCPConnected**  : this Boolean will show you the state of the TCP connection to the broker

**mqttData._state** : will show you the connection status of the MQtt connection to the broker. Check for value > 0 to check if MQTT is connected. I recommend to check this status before trying to send a message.

**mqttData.mqttErrorCode**  : will hold the last MQtt error code

**mqttData.tcp_sendBufferFull** : will be true if the send buffer is full. Important: this also indicates that the last message could not be added to the send buffer, the last message therefore was discarded. To avoid problems with the send buffer, give it an appropriate size to hold multiple messages and also only send if mqttData._state > 0. 

# Example

Included is an example application function block (FB70) that is typically called from OB1.
Inputs for this block can trigger a MQTT broker connect, publish a message or subscribe to a MQTT channel.

There is also some example code for calculating a payload CRC to check the data integrity.  The used CRC_GEN function can be found in the oscat.de PLC library.

# Notes
Porting C++ to Siemens SCL has taught many differences between systems. Differences include:

- The inability to make blocking calls in SCL.
- No dynamically allocated memory.

One reason for this is a PLC is a real-time system. A consequence is that the entire program code, from the first to last line, *must* complete in a certain amount of time. (Typically <10ms)
The workaround I make is using a FB with a state machine that can wait for a procedure to finish before moving on.

After working with SCL for a while it feels that the language is primitive in many ways in comparison with most PC languages.
I realise that some of it is because of program limitation for safety and resource conservation, but general program principles like DRY, OOP, etc. are sometimes difficult to achieve if not impossible.

Another annoying difference is the idea of having symbol entries for symbolic block names. On one side I want to refactor and break behaviour up in more functions like in general programming.
But in Siemens I don't want to create a new symbol entry for each FC that I make. Also, this is probably not encouraged in PLC programming anyway as it increases the maximum size of the call stack in every scan cycle and increases the scan time.
This makes it difficult to make small sized program code that is also performant.

Furthermore, the low-level programming and memory limitation reminds of microcontroller programming.
