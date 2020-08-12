=========
Firmware
=========

The firmware for the tool access project is written primarily in Arduino flavoured C++. The RTOS "Real Time Operating System" (based on freeRTOS) that the Arduino
core for the ESP32 rides on top of is utilized in the RTOS branch for asynchronous task handling. Authentication of RFID cards is carried out at the server level
rather than at the tool level where the card is read. MQTT is utilized for asynchronous communication with the authentication sever. 

Linear vs RTOS Branches
--------------------------

The firmware for the tool access project has two branches as of |today|: Linear and RTOS. 

The linear branch is "standard" linear Arduino code. It can't query a database and instead uses a hardcoded UID's as a "stub" for authorization of cards. 

The RTOS branch utilizes the  RTOS running underneath the Arduino layer for more graceful task handling. The Arduino framework is still utilized to interact 
with peripherals but is wrapped in RTOS Tasks. This branch communicates with an MQTT broker via WiFi for authentication of RFID cards.


Learning RTOS on the ESP32
-----------------------------
`ESP32 Meet-up FreeRTOS <https://www.youtube.com/watch?v=E9FY-IOvC3Q>`_ I watched this on 1.75X to give me the basic gist of the RTOS API

`Mastering the FreeRTOS Real Time Kernel <https://www.freertos.org/wp-content/uploads/2018/07/161204_Mastering_the_FreeRTOS_Real_Time_Kernel-A_Hands-On_Tutorial_Guide.pdf>`_ This is an official freeRTOS resource. The ESP RTOS is based on it but is not exactly the same. I recommend skipping striaght to the section on Task Management.

`ESP-IDF FreeRTOS SMP Changes <https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/freertos-smp.html>`_ Espressif's documentation for the differences bewteen ESP32's RTOS and vanilla freeRTOS

`ESP32 FreeRTOS API Reference <https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/system/freertos.html>`_ The Espressif documentation for their flavour of FreeRTOS

.. warning::
   Do not solely utilizes the freeRTOS documentation without referencing Espressif's ESP32 specific RTOS documentation. 
   There are differences in how the two are implemented.


Tool Access Data Structures
-----------------------------

One of the constraints of working with RTOS is that tasks cannot be passed any number of variables like a regular function. Instead they take a single void point (void *).
RTOS has a number of tools for passing data between tasks such as queues, stream, and message buffers. Instead of using these more complex tools the Tool Access Firmware
simply passes the address of nested structures via the void pointer at task creation. This allows for the tasks to continue passing down the struct pointer to 
help functions they may need to call.

These nested structures are described in ```toolAccessRTOS.h```

.. code-block:: C++
   :linenos:

    // Struct for holding UID and manipulating of UID data type
    typedef struct {      
        byte uid_ByteBuffer[10]; // Holds UID read out of MFRC522 library
        byte uid_ByteLength;     // Holds size of uid_ByteBuffer
        byte uid_Test[10];
        // Holds the string version of the uid byte living in the mfrc522 struct
        size_t uidStrLen; // Holds the length (not size) of uidStr
        char uidStr[31]; // Biggest possible UID is 10bytes * 3 (because we : separate, eg. 0xFF:etc) + 1 (NULL) = 31
        // Counter used in detecting card collisions 
        char collCounter;     
    }cardParams;

    // Stuct for passing desired LED parameters
    typedef struct{ 
        int led; // # of LED in FastLED array
        int time; // blink rate
        bool blink; // whether to blink or not
        CRGB myColour; 
    } LEDParams;

    // Struct for holding timer information
    typedef struct{
        uint32_t ms_timeLast = 0;  // millisecond timer for holding last time was in a function.
        uint32_t ms_timeOut = 0; // Holds ms_timeOut count down for when card is removed.
        uint32_t ms_tooluseTime = 0; // Holds time relay was opened on timeout and therefore the time the tool was in use
        char min_tooluseTime = 0;
        uint32_t ms_relayStart = 0; // Holds time (milliseconds) at initial closing of relay
        uint32_t ms_wifi_outage = 0; // ?
    } Timers;

    // Struct for holding cardParams, LEDParams, and Timers
    typedef struct{
        LEDParams LEDParams0; // Parameters for LED0
        LEDParams LEDParams1;  // Parameters for LED1
        Timers timers;         // Instance Timers struct to hold all of our relevant timers
        cardParams card;      // Holds info about read card  
    } metaStruct;


metaStruct contains declared members of the other structs. It functions as a container who's address we may pass via the void pointer.

**Passing our structs via the void pointer**

.. code-block:: C++
   :linenos:
   
   // Example RTOS Tasks
   void pollNewTask (void *params){
         /* We may transfer our pointer address 
         from our void pointer to a new variable via casting*/
         metaStruct *progParams = (metaStruct*) params; 

        // We may now access our struct members like so 
        progParams->card.uidByte; // Only progParams is a pointer requiring the -> operator. 
        // After accessing via point we must use the . operator 
   }

   void setup(){
        // All code we wish to only run once is still placed in void setup

        // We declare a member of metaStruct, our container
        metaStruct progParams;

        // If any of our struct variables require initialization we do so
        progParams.LEDParams0.myColour = CRGB::Black; // We want our LEDs to start off
        progParams.LEDParams0.led = 0; // Notice that because we are still in setup we access
        progParams.LEDParams1.led = 1; // with the . operator all the way down our structs
        progParams.LEDParams0.blink = 0;
        progParams.LEDParams1.blink = 0;
   
        // Here we pass the address of just progParams to our pollNew Task via the void * parameter
        xTaskCreatePinnedToCore(pollNewTask, "pollNewTask", 2048, &progParams, 1, &pollNewHandle, 1);
   }

   void loop(){
   // Loop is not used when working with the RTOS 
   }
   


Peripheral Interactions
-----------------------
As of |today| the tool access system has a set list of peripherals:
1. 2X COM12999 Addressable LEDs
2. 1X MFRC522 RFID Module
3. 1X Power relay and associated driver circuit 

RFID - MFRC522 Module
^^^^^^^^^^^^^^^^^^^^^^

Technical documentation:

`List of status codes and types <https://docu.byzance.cz/hardware-a-programovani/programovani-hw/knihovny/mfrc522>`_

`Miguelbalboa write up <https://diy.waziup.io/assets/src/sketch/libraries/MFRC522/doc/rfidmifare.pdf>`_

`MIFARE ISO/IEC 14443 PICC Selection <https://www.nxp.com/docs/en/application-note/AN10834.pdf>`_


The cheap and ubiquitous MFRC522 RFID module utilizes the NPX MFRC522 chip which is capable of a great deal more than it is used for in this code.
All we need it to do is detect a MIFARE card and read its UID. The server side of this project can associate UIDs with specific members if need be. 

Library\: `Miguelbaoboa MFR522 Arduino Library <https://github.com/miguelbalboa/rfid>`_. This library if no longer maintained by the original author but instead by community support.


.. warning::
   The ability to detect collisions (>1 card in RF field) is not functional on many of the cheap/ubiquitous RC522 modules available. 
   This is even called out in the Miguelbaoboa's RFID library where he speculates that it may be due to poor antenna design. Because of this the collision detection
   implemented in RFID library as per the datasheet recommendations does not function as it should.


Glossary of MIFARE Technical Abbreviations & Terms 

.. glossary::
   PICC
      Proximity Integrated Circuit Card

   PCD
      Proximity Coupling Device

   UID
      Unique IDentifier number set by factory on each PICC (bytes 0..6 of block 0) may be be 4-10 bytes.

   SAK
      Select ACknowledge

   REQA/B
      REQuest, type A/B, polling command sent out by PCD ~5ms.

   ATQA/B
       Answer To reQuest, type A/B, response to REQ returned by PICC if present in RF field.

   WUPA
      Wake-Up Protocol, type A

   ACK
       ACKnowledge
       
   NAK
       Not AcKnowledge

   POR
      Power On Reset

   CRC
      Cyclic Redundancy Check

Manufacturer Recommended Control Loop
""""""""""""""""""""""""""""""""""""""""

!`7743dd5b0abc9f8b621a3c901add9fc5.png <:/3c61707ed3bf49cea1815d1afb6ff0cf>`_
Card polling block diagram from `MIFARE ISO/IEC 14443 PICC Selection <https://www.nxp.com/docs/en/application-note/AN10834.pdf>`_ (Pg 5). ATQA/B - Answer to Request. REQA/B - Request Command . A/B - Type A or B card, protocol must be compatible with both and therefore polls both. 

!`c985f20daccaaf4976a0782d38ce6652.png <:/55ef4d97e75542fba10bf009d13c60f1>`_
Card selection block diagram (without MAD - MIFARE Multiple Application Directory) from `MIFARE ISO/IEC 14443 PICC Selection <https://www.nxp.com/docs/en/application-note/AN10834.pdf>`_ (Pg 10). N = # of cards in RF field. Reactivate halted cards via WUPA (Wake Up Command).

Although represented in a slightly confusing manner the control flow diagrams provided by NPX (see above) layout how to interact with the RFID hardware. 

1. We poll for new cards. This is done by the MFRC522 broadcasts an REQA/B command. If a card is present in the RF field (and has had 5ms to boot) it will respond with a ATQA/B command.
2. We proceed to authenticating the card. This is where the UID is read.
3. If a collision occurs (ie. >1 card in the RF field).



Control Loop Utilized by Tool Access Project 
"""""""""""""""""""""""""""""""""""""""""""""

**States**

The control flow for the RFID hardware is state based. Our ESP should only close the relay under certain circumstances.
The states and the transitions between those states are a result of the number of RFID cards present in the modules RF field.

1. *No cards present* - in this state we poll for the arrival of new cards.
2. *One card present* - in this state we have detected a card. We must authorize it if the relay is to be closed. We must also shift from polling for new cards 
   to polling for the continued presence of our detected card and polling for a collision event.
3. *Collision (>1 card present)* - In this state we have detected a collision and we transition to Timeout state. Why is this done? We can detect the resolution 
   of a collision ie. one of the cards being removed however in the case of an unauthorized card colliding with an authorized one tailing in can be achieved by 
   careful removal of the authorized card. 
4. *timingOut* - In this state a timer is run down because either a collision has occurred or a card has been removed. This state can be exited by introducing
   a new card to reader or on expiration of the timer. Therefore we may think of it as occurring concurrently with the no cards present state. 

.. figure:: ./images/toolAccessStates.png
   :align: center
   :alt: State diagram for MFRC522 Hardware

   All state transitions are conditional except for Collision goes to timingOut which occurs unconditionally.

This state diagram holds true for both the Linear and the RTOS branches of the code. Authorization step omitted for clarity.

.. Note::
   The unconditional transition from the Collision state to the timingOut state is necessary due to the MFRC522 modules returning TIMEOUT status codes instead of
   COLLISION status code in the event of a collision. This does not prevent us from detect collisions but rather detecting how a collision is resolve.
   Therefore the decision was made to push all collisions to the timingOut state. 

State Transitions in RTOS
''''''''''''''''''''''''''''


COM12999 - Addressable LEDs
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
 
Library \: `FastLED <https://github.com/FastLED/FastLED>`_

blinky LED Task

MQTT
-------

Library\: `Async MQQT Client <https://github.com/marvinroger/async-mqtt-client>`_


Proposed Topic Structure
^^^^^^^^^^^^^^^^^^^^^^^^^^^
Technical documentation\:
`MQTT Topics & Best Practices <https://www.hivemq.com/blog/mqtt-essentials-part-5-mqtt-topics-best-practices/>`_

This is the proposed MQTT Topic Structure

.. code-block:: C++
   :linenos:
   
   // + single level wildcard
   // # multi level wildcard
   // wildcards may be used to subscribe to topics only not publish

   // Whoami - would allows tool to append hardcoded MQTT topics to include workshop/toolalias 
   tool/MAC // payload: workshop, toolalias

   // Workshops level topics
   tools/woodshop/toolalias
   tools/fasbshop/toolalias
   tools/machineshop/toolalias
   tools/electronics/toolalias
   tools/sewing/toolalias
   
   // Authorization topics
   tools/+/+/auth/req // payloads: UID
   tools/+/+/auth/rsp // payloads: auth|denied|seekiosk
   tools/+/+/auth/eou // uid
   
   // Estop topics
   tools/estop   // Makerspace level
   tools/+/estop // Workshop level
   tools/+/+/estop // Tool level
   
   // Logging topics
   tools/+/+/logs/ // payload: status
   tools/+/+/logs/status/rsp // payload: true, sizeoflog|false
   tools/+/+/logs/send // payload: req
   tools/+/+/logs/send //payload: JSON document holding logs?


# Roadmap to Further Development
## Optimization
### Interrupt functionality of the MFRC522 module
The MFRC522 chip supports interrupts generated on pin 5. The PCB design has left this pin unconnect so that is may be soldered to one of the ESP pins if desired. 

If this is to be pursued RTOS function calls will need to be changed to their ISR safe equivalents.

### Shrinking program size for OTA
For the over the air updates functionality to be used our program must occupty <50% of flash memory. As of 2020/08/07 it occupies ~59%. Additioanlly as part of the OTA process logs from tools will havbe to be requested and trasnmitted before the OTA is initiated as this process will likely overwrite the SPIFFS partition.

## Future features
## Potential Pitfalls

