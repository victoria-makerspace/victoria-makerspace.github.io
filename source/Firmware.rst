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

Getting Setup to Code with the ESP32
-------------------------------------

There are three main routes to getting coding on the ESP32.

1. Through the Arduino index (most beginner friendly, can code in Arduino flavoured C++)

2. VSCode + Espressif IDF (for advanced users/C literate coders)

3. VSCode + Platformio (Intermediate but with its own share of headaches)

One issue than can be encountered in going this route is the compiler finding two WiFi.h files particularly if you already have the Arduino IDE installed.
One of these is located in the directory in which the Arduino IDE is installed in a folder named libraries. This is distinct from the libraries folder where external
libararies are installed. This is the Arduino core libraries.
The other WiFi.h is located in the platformio directory under ``packages\frameswork-arduinoespressif32\libraries``. This is the WiFi.h you want to use.



Learning RTOS on the ESP32
-----------------------------

1. `ESP32 Meet-up FreeRTOS <https://www.youtube.com/watch?v=E9FY-IOvC3Q>`_ I watched this on 1.75X to give me the basic gist of the RTOS API
2. `Mastering the FreeRTOS Real Time Kernel <https://www.freertos.org/wp-content/uploads/2018/07/161204_Mastering_the_FreeRTOS_Real_Time_Kernel-A_Hands-On_Tutorial_Guide.pdf>`_ This is an official freeRTOS resource. The ESP RTOS is based on it but is not exactly the same. I recommend skipping striaght to the section on Task Management.
3. `ESP-IDF FreeRTOS SMP Changes <https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/freertos-smp.html>`_ Espressif's documentation for the differences between ESP32's RTOS and vanilla freeRTOS
4. `ESP32 FreeRTOS API Reference <https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/system/freertos.html>`_ The Espressif documentation for their flavour of FreeRTOS
5  `How to pass pointers to Software Timers <https://www.freertos.org/FreeRTOS_Support_Forum_Archive/May_2016/freertos_xTimerCreate_vTimerCallback_b3906645j.html>`_ Passing pointers pointers to software timer callback 
functions is also possible but is not made as clear by the official documentation.

.. warning::
   Do not solely utilizes the freeRTOS documentation without referencing Espressif's ESP32 specific RTOS documentation. 
   There are differences in how the two are implemented.


Tool Access Configuration and Initialization 
----------------------------------------------

All hardware pin assignments, timer periods, MQTT IP addresses, etc can be found in the ``toolAccessRTOS.h`` and ``toolAccessLinear.h`` respectively. All assignments are made via defines.  

All timing periods in the ``toolAccessRTOS.h`` are made as a wrap of the ``pdMS_TO_TICKS(period_ms)`` RTOS macro which converts a desired period expressed in milliseconds to RTOS ticks.  

Tool Access Data Structures
-----------------------------

One of the constraints of working with RTOS is that tasks cannot be passed any number of variables like a regular function. Instead they take a single void point (void *).
RTOS has a number of tools for passing data between tasks such as queues, stream, and message buffers. Instead of using these more complex tools the Tool Access Firmware
simply passes the address of nested structures via the void pointer at task creation. This allows for the tasks to continue passing down the struct pointer to 
help functions they may need to call.

.. code-block:: C++
   :linenos:
   :caption: Tool Access Data Structs defined in ``toolAccessRTOS.h``

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

    // Struct for holding cardParams, LEDParams, and Timers
    typedef struct{
        LEDParams LEDParams0; // Parameters for LED0
        LEDParams LEDParams1;  // Parameters for LED1
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

.. note:: 

   Software timer callback functions can also have pointers passed through its ``void * pvTimerID`` parameter in the ``xTimerCreate()`` call. 
   The RTOS documentation does not explicitly state this like it does for tasks creation. 
   

Peripheral Interactions
-----------------------
This section of the documentation focuses on the code I wrote to interact with the peripherals not on explaining how those peripherals work. Where a greater understanding of
the peripheral may be necessary in order to understand how my code works is the MFRC522 module which has its own page here.

RFID - MFRC522 Module
^^^^^^^^^^^^^^^^^^^^^^

The cheap and ubiquitous MFRC522 RFID module utilizes the NPX MFRC522 chip which is capable of a great deal more than it is used for in this project.
For our purposes all we need it to do is detect a MIFARE card and read it's UID. The server side of this project can associate UIDs with specific members. 

Technical documentation:

1. `List of status codes and types <https://docu.byzance.cz/hardware-a-programovani/programovani-hw/knihovny/mfrc522>`_
2. `Mario Capurso's write up using MFRC522 Arduino library <https://diy.waziup.io/assets/src/sketch/libraries/MFRC522/doc/rfidmifare.pdf>`_
3. `MFRC522 Datasheet <https://www.nxp.com/docs/en/data-sheet/MFRC522.pdf>`_
4. `MIFARE ISO/IEC 14443 PICC Selection <https://www.nxp.com/docs/en/application-note/AN10834.pdf>`_ 

Library\: `Miguelbaoboa MFR522 Arduino Library <https://github.com/miguelbalboa/rfid>`_. This library if no longer maintained by the original author but instead by community support.


.. warning::
   The ability to detect collisions (>1 card in RF field) is not functional on many of the cheap/ubiquitous RC522 modules available. 
   This is even called out in the Miguelbaoboa's RFID library where he speculates that it may be due to poor antenna design. Because of this the collision detection
   implemented in RFID library as per the datasheet recommendations does not function as it should.

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


**RFID States Diagram**

.. mermaid's :caption: directive places the caption on the bottom not the top

.. mermaid::
   :align: center 
   :caption: All state transitions are conditional except for Collision goes to timingOut which occurs unconditionally. Authorization step omitted for clarity.

   stateDiagram
   [*]-->noCard
   noCard --> oneCard: Card detected
   
   state noCard{
   	openRelay
   }
   
   state oneCard{
   	closeRelay
   }
   
   state oneCard <<fork>>
   oneCard-->Collision: >1 card present
   oneCard-->timingOut: Card removed
   
   
   Collision-->timingOut
   
   state timingOut{
   	startTimer
   }
   
   state timingOut <<fork>>
   timingOut-->oneCard: Card detected
   timingOut-->noCard: Timeout

   
This state diagram holds true for both the Linear and the RTOS branches of the code. The states and state transitions are simply handled differently. In the linear
branches the states are tracked via boolean flag variables and transitions are made via conditional checks against those flags. In the RTOS branch this is done via 
EventGroups.

.. Note::
   The unconditional transition from the Collision state to the timingOut state is necessary due to the MFRC522 modules returning TIMEOUT status codes instead of
   COLLISION status code in the event of a collision. This does not prevent us from detect collisions but rather detecting how a collision is resolved.
   See MFRC522 primer for more detail. 

State Transitions in RTOS
""""""""""""""""""""""""""""

In the RTOS branch of the code states are tracked via the EventBits contained within the EventGroup ``rfidStatesGroup``. The EventBits are interacted with via RTOS API calls
and macros defined in ``toolAccessRTOS.h``.

.. code-block:: C++
   :linenos:
   :caption: EventBit macros found in ``toolAccessRTOS.h``

   // Event group macros
   #define CARD_BIT_0 ( 1 << 0 )
   #define AUTH_BIT_1 ( 1 << 1 )
   #define RELAY_BIT_2 ( 1 << 2 )
   #define TIMEOUT_BIT_3 ( 1 << 3 )
   #define COLL_BIT_4 ( 1 << 4 )
   #define ESTOPFIRE_BIT_5 ( 1 << 5 )
   #define ESTOPCLEAR_BIT_6 ( 1 << 6 )
   #define WIFIOUT_BIT_7 ( 1 << 7 )

Not all of the EventBits are utilized to make state transitions but are set or cleared according to the state they are named for in the event that they may be used for state transitions in the future.

The four main RTOS API calls used to interact with the Event bits are

.. code-block:: C++
   :linenos:

   xEventGroupClearBits(EventGroupHandle_t xEventGroup, const EventBits_t uxBitsToClear)); // Clears specified bits
   xEventGroupSetBits(rfidStatesGroup, (CARD_BIT_0|AUTH_BIT_1)); // Sets specified bits
   EventBits_t xEventGroupWaitBits(const EventGroupHandle_t xEventGroup,const EventBits_t uxBitsToWaitFor,const BaseType_t xClearOnExit,const BaseType_t xWaitForAllBits,TickType_t xTicksToWait);
   xEventGroupGetBits(rfidStatesGroup); // Checks value held in rfidStatesGroup

Line one shows xEventGroupClearBits as the definition while line 2 shows xEventGroupSetBits as an actual call (they expect the same parameters).

.. important::
   ``CARD_BIT_0|AUTH_BIT_1`` are passed with **bitwise OR** because we are creating a bitmask as an operator on the binary value contained within 
   rfidStatesGroup.

Line 3 once again shows a formal definition. xEventGroupWaitBits is the call used to gate state transitions. It blocks a task (not the processor) until the specified bits 
are set. It cannot be used to check for being cleared.  Notice that it can be configured to block until both specified bits are set or either bit is set. Additionally
it can clear the bits it checks on returning. 

Line 4 shows how the value held in an EventGroup could be checked if a conditional operation needs to be done outside of the RTOS API calls such as ``xEventGroupWaitBits``.

.. warning::
   Setting EvetBits can unblock multiple tasks at once. This can result in nondeterministic behaviour if care is not taken.

**RTOS States Diagram**



**RTOS RFID Tasks**

Library\: `MFRC522 <https://github.com/makerspaceleiden/rfid>`_ - This version of the MFRC522 library is a rewrite of the Miguel Balboa library by members of the Leiden 
Makerspace.

.. pollNewTask () 
   waitBits = none
   if new card
   setBits = CARD_BIT_0
     if check bits == 129
     logging 
     setBits = AUTH_BIT_1
     else
     pub uid for auth
   vTaskSuspend(NULL)

.. topic:: ``pollNewTask (void *params)``

   ``pollNewTask()`` is the entry point for the RFID control loop. Once it detects a new card it suspends itself. It is very important it is resumed at the appropriate places for the control loop to keep
   functioning (ie. once we have entered ``timeoutTask``). However, this also means that we can disable this task as a means of suspending the RFID functionality, see the EstopFire and 
   EstopClear tasks for more details.

   This task also receives a task notification. This notification is made from ``onMqttMessage()`` when a card is denied by the server. The LEDParam structs are out of scope of the MQTT callback functions 
   of the AsynMQTTClient library and I could not figure out a way of passing them the value. Instead ``pollNewTask()``, which is also resumed when a card is denied, is notified so that it may change the LEDParams
   to blinking red.

   When WiFi/MQTT is connected only CARD_BIT_0 is set in ``pollNewTask()`` and AUTH_BIT_1 is set elsewhere in ``onMqttMessgae()`` based the MQTT payload of ``tool/auth/rsp``. 
   When a WiFi or MQTT disconnection event occurs WIFIOUT_BIT_7 is set. This is detected by bitmasking ``xEventGroupGetBits(rfidStatesGroup)`` to check the 7th bit (WIFIOUT_BIT_7)
   to see if it has been set. If the WIFIOUT_BIT_7 has been set both CARD_BIT_0 and AUTH_BIT_1 are set within ``pollNewTask()`` without server authorization. 
   This is where future implementation for flash memory logging should be placed.

   RTOS API calls summary

   +---------------------+------------------------------+------------------------------+-------------------+-------------------------------+--------------+
   | WaitBits()          | SetBits()                    | GetBits ()                   | vTaskDelay()      | TaskNotifyTake()              | vTaskSuspend |
   +=====================+==============================+==============================+===================+===============================+==============+
   | N/A                 | If WIFIOUT_BIT_7 not set     |                              | MS_POLL_TIMER     | notificationValue == 1        | NULL         |  
   |                     | CARD_BIT_0                   |                              |                   | toggle LEDparams   \n         |              |
   +---------------------+------------------------------+                              +                   + Red                           +              |
   | N/A                 | If WIFIOUT_BIT_7 is set      | Used to check WIFIOUT_BIT_7  |                   | blinkTemp                     |              |
   |                     | else CARD_BIT_0 | AUTH_BIT_1 |                              |                   |                               |              |
   +---------------------+------------------------------+------------------------------+-------------------+-------------------------------+--------------+


.. closeRelayTask()
   waitBits = CARD_BIT_0 | AUTH_BIT_1
   setBits = RELAY_BIT_2
   vTaskSuspend ( NULL )

.. topic:: ``closeRelayTask(void *params)``

   Unblocks once CARD_BIT_0 and AUTH_BIT_1 are set. It closes the relay, sets the RELAY_BIT_2 (not currently used for anything), changes the LEDs to a solid green to 
   indicate this to the user. The task then suspends itself once all of this has been achieved. This task is resumed in ``timeoutTask()``.

   +-----------------------+---------------+------------------------------+-----------------+
   | WaitBits()            | SetBits()     | vTaskSuspend ()              | vTaskResume ()  +
   +=======================+===============+==============================+=================+
   | CARD_BIT_0|AUTH_BIT_1 | RELAY_BIT_2   | NULL                         | blinkLEDHand0/1 | 
   +-----------------------+---------------+------------------------------+-----------------+
   
.. pollPresTask ()
   waitBits = CARD_BIT_0 | AUTH_BIT_1
   SemaphoreGive SPIMutexHandle
   vTaskDelay()
      if present
         Halt
      else
         clearBits = CARD_BIT_0 | AUTH_BIT_1
         setBits = TIMEOUT_BIT_3
   SemaphoreTake SPIMutexHandle

.. topic:: ``pollPresTask(void *params)``

   Unblocks when CARD_BIT_0 and AUTH_BIT_1 are set. Because both ``pollPresTask``\  and ``collPollTask`` occur in the same state and both utilize the SPI bus a Mutex is given/taken by the two Tasks. 
   If ``pollPres()`` detects that a card has left the RF field the CARD_BIT_0 and AUTH_BIT_1 is cleared then the TIMEOUT_BIT_3 is set. This unblocks the ``timeoutTask()`` while placing both
   ``pollPresTask()`` and ``collPollTask()`` back in the blocked state. 

   LEDs are set to blink blue when a card removal is detected.


   +-------------------------+---------------+-------------------------+----------------------+-----------------------+-----------------+
   | WaitBits()              | SetBits()     | ClearBits ()            | vTaskDelay()         | SemaphoreGive/Take () | vTaskResume ()  +
   +=========================+===============+=========================+======================+=======================+=================+
   | CARD_BIT_0 | AUTH_BIT_1 | TIMEOUT_BIT_3 | CARD_BIT_0 | AUTH_BIT_1 | MS_POLL_TIMER_PERIOD | SPIMutexHandle        | blinkLEDHand0/1 |
   +-------------------------+---------------+-------------------------+----------------------+-----------------------+-----------------+
   

.. collPollTask ()
   waitBits = CARD_BIT_0 | AUTH_BIT_1
   SemaphoreTake SPIMutexHandle
      if coll 
         clearBits = CARD_BIT_0 |  AUTH_BIT_1
         setBits = TIMEOUT_BIT_3 | COLL_BIT_4
      else
      // do nothing
   SemaphoreGive SPIMutexHandle

.. topic:: ``collPollTask(void *8*params)``

   Unblocks when CARD_BIT_0 and AUTH_BIT_1 are set. It gives/takes a mutes with ``pollPresTask()`` to avoid conflicting access of the SPI bus. If a collision is detected by ``collPolling()``
   CARD_BIT_0 | AUTH_BIT_1 are cleared and the TIMEOUT_BIT_3 | COLL_BIT_4 are set in order to execute a transition to the timeout state. 

   LEDs are set to blink purple to indicate a collision has been detected.

   +-------------------------+----------------------------+-------------------------+----------------------+-----------------------+-----------------+
   | WaitBits()              | SetBits()                  | ClearBits ()            | vTaskDelay()         | SemaphoreGive/Take () | vTaskResume ()  +
   +=========================+============================+=========================+======================+=======================+=================+
   | CARD_BIT_0 | AUTH_BIT_1 | TIMEOUT_BIT_3 | COLL_BIT_4 | CARD_BIT_0 | AUTH_BIT_1 | ?                    | SPIMutexHandle        | blinkLEDHand0/1 |
   +-------------------------+----------------------------+-------------------------+----------------------+-----------------------+-----------------+

.. timeoutTask ()
   waitBits = TIMEOUT_BIT_3
   vTaskDelay MS_TIMEOUT_PERIOD
   waitBits = TIMEOUT_BIT_3 // two step block
   pub eou/to
   digitalWrite RelayOpen
   clearBits = TIMEOUT_BIT_3 | RELAY_BIT_2

.. topic:: ``timeoutTask (void *params)``
   
   Unblocks on the TIMEOUT_BIT_3. The first thing that is done is publishing to the MQTT ``rfid/auth/eou`` (end of use) topic. This is done before
   ``pollNewTask()`` is resumed so that it cannot be interrupted by introduction of a new card. Next ``closeRelayTask`` is resumed so that if a new card is introduced
   the relay can be closed. A bitmask operation is carried out against ``xEventGroupGetBits`` to determine if both TIMEOUT_BIT_3 and COLL_BIT_4 are set.

   If just TIMEOUT_BIT_3 is set ``pollNewTask()`` is resumed so that a new card can be introduced and authorized without the relay being opened.

   If TIMEOUT_BIT_3 and COLL_BIT_4 is set the full timeout is enforced and the relay is re-opened before ``pollNewTask()`` is resumed.

   When the full timeout occurs RELAY_BIT_2, TIMEOUT_BIT_3, and COLL_BIT_4 are all cleared.

   +-------------------------+-----------------------------+--------------------------------+-------------------+-----------------+
   | WaitBits()              | ClearBits()                 | GetBits ()                     | vTaskDelay()      | vTaskResume ()  +
   +=========================+=============================+================================+===================+=================+
   | TIMEOUT_BIT_3           | RELAY_BIT_2 | TIMEOUT_BIT_3 | Used to check if TIMEOUT_BIT_3 | MS_TIMEOUT_PERIOD | blinkLEDHand0/1 |
   |                         |                             +                                +                   +                 +                                
   |                         |                             | and/or COLL_BIT_4 is set       |                   | pollNewHandle   |
   +-------------------------+-----------------------------+--------------------------------+-------------------+-----------------+

**EStop Tasks**

.. Ignore for now
   eStopSetTask ()
   waitBits = ESTOPFIRE_BIT_5

.. topic:: ``eStopFireTask (void *params)``

   Unblocks on ESTOPFIRE_BIT_5 which is set in ``onMqttMessage()`` on receipt of ``rfid/estop/fire``. A ``xEventGroupGetBits()`` is used to check the state of the hardware. If
   no bits are set ``pollNewTask()`` is suspended cutting off the access point to the RFID functionality. If any of the EventBits are set all the bits are cleared, the relay is 
   opened and then ``pollNewTask()`` is suspended. The LEDs are then set to flash yellow.

   The ESTOPSET_BIT_6 auto clears on the the successful ``xEventGroupWaitBits()``. 

   +-------------------------+--------------------------------+-------------------------------------------------------------------+-----------------+
   | WaitBits()              | GetBits ()                     | ClearBits()                                                       | vTaskResume ()  +
   +=========================+================================+===================================================================+=================+
   | ESTOPFIRE_BIT_5         | If no bits set                 | N/A                                                               | blinkLEDHand0/1 |
   +                         +--------------------------------+-------------------------------------------------------------------+                 +
   |                         | If bit other than ESTOPbits set| CARD_BIT_0 | AUTH_BIT_1 | RELAY_BIT_2 | TIMEOUT_BIT_3 | COLL_BIT_4| pollNewHandle   |
   +-------------------------+--------------------------------+-------------------------------------------------------------------+-----------------+

.. eStopClearTask ()
   waitBits = ESTOPCLEAR_BIT_6

.. topic:: ``eStopClearTask (void *params)``

   Unblocks on ESTOPCLEAR_BIT_6 which is set in ``onMqttMessage()`` on receipt of ``rfid/estop/fire``. This task simply turns off the LEDs and resumes ``pollNewTask()``.
   The ESTOPCLEAR_BIT_6 auto clears on the the successful ``xEventGroupWaitBits()``.
   
   +--------------------------+-----------------+
   | WaitBits()               | vTaskResume ()  +
   +==========================+=================+
   | ESTOPCLEAR_BIT_5         | blinkLEDHand0/1 |
   +                          +                 +
   |                          | pollNewHandle   |
   +--------------------------+-----------------+



COM12999 - Addressable LEDs
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Control of the COM12999 addressable LEDs is done via the FastLED library, a single RTOS task definition ``blinkyLEDTask``.

Library \: `FastLED <https://github.com/FastLED/FastLED>`_

blinky LED Task

.. code-block:: C++
   :caption: blinky LED Task
   :linenos:

   void blinkyLED (void *params){

      LEDParams *l = (LEDParams*)params; // Dumping our struct parameters into task instance of LEDParams via casting of void *params to LEDParams

  
      for (;;){ // Infinite loop required for RTOS tasks b/c if allowed to return they would delete

         // Branch based on blink status  
         if(l->blink){                             // if blink flag has been set
            leds[l->led] = CRGB::Black;            // Set LED specified in passed params to OFF state 
            FastLED.show();                        // Toggle the LED state to new
            vTaskDelay(pdMS_TO_TICKS(l->time));    // RTOS delay blocks task not processor

            leds[l->led] = l->myColour;            // Set LED specified in passed params to the colour specified in same passed params
            FastLED.show();
            vTaskDelay(pdMS_TO_TICKS(l->time));   // Set delay time according to passed param AND div by port TICK period ms
         }  
         else{ 
            leds[l->led] = l->myColour;            // Set LED specified in passed params to the colour specified in same passed params
            FastLED.show();                        // Show it 
            vTaskSuspend ( NULL );                 // Suspend ourselves since blink = false therefore the task need not keep running 
         }
      }
   }

Where the LED control tasks differ slightly from the other RTOS tasks is at creation they are passed the address of their parameter struct rather than the address of the metaStruct. The LED tasks
need only access to the LED parameters while other tasks needs access to their own parameters and the LED parameters.

.. code-block:: C++
   :caption: LED Task Creation in void setup ()

   //Task creation
   xTaskCreatePinnedToCore(blinkyLED, "blinkLED", 1024, &progParams.LEDParams0, 1, &blinkLEDHandle0, 1);
   xTaskCreatePinnedToCore(blinkyLED, "blinkLED1", 1024, &progParams.LEDParams1, 1, &blinkLEDHandle1, 1);
   
Manipulation of the LEDs can then be achieve simply by changing the values held in ``LEDParams0`` and ``LEDParams1`` via the void pointer. 

.. note::
   If the previous state of an LED was ``blink == 0`` then its respective LEDTask will have to be resumed.

.. code-block::
   :caption: Example LED manipulation

   void exampleTask (void *params){
   metaStruct *progParams = (metaStruct*) params;
      
      for(;;){
       progParams->LEDParams0.myColour = CRGB::Red;     // Set LED0 to Red
       progParams->LEDParams1.myColour = CRGB::Purple;  // Set LED1 to Purple
       progParams->LEDParams0.blink = 0;                // Set LED0 to continuous one 
       progParams->LEDParams1.blink = 1;                // Set LED1 to blink
       // Without changing progParams->LEDParams1.time LED1 will blink at whatever period was last defined there

      vTaskResume(blinkLEDHandle0); // Resume the LED task
      vTaskResume(blinkLEDHandle1); // Resume the LED task
      }
   }

MQTT
-------

The ESP32 side of the MQTT transitions are handled using the Async MQTT library and modified versions of the functions written in the ``FullyFeatured-ESP32.ino`` example included with the library.

Technical documentation\:
`MQTT Topics & Best Practices <https://www.hivemq.com/blog/mqtt-essentials-part-5-mqtt-topics-best-practices/>`_
Library\: `Async MQTT Client <https://github.com/marvinroger/async-mqtt-client>`_
Library dependancy\: `AsyncTCP <https://github.com/me-no-dev/AsyncTCP>`_

.. Marvinroger seems to have his own version of the AsyncTCP library for some reason

Current MQTT Structure
^^^^^^^^^^^^^^^^^^^^^^^^

The current MQTT structure (as of 2020/08/17) is relatively simple for the sake of prototyping.

.. code-block:: C++
   :caption: Current MQTT Structure 

   // Publishes
   rfid/auth/req // payload: uid
   rfid/auth/eou // payload: uid 
   
   // Subscriptions
   rfid/estop //payload: fire|clear
   rfid/auth/rsp // payloads: auth|denied|seekiosk

MQTT functions
^^^^^^^^^^^^^^^^

As mentioned above the MQTT functions are those included in the ``FullyFeatured-ESP32.ino`` example included with the library. Most of them remain unmodified, the exceptions to this are
``connectToWifi()``,  ``WiFiEvent()``, ``onMqttConnect()``, and ``onMqttMessage()`` . As such I will only discuss these functions.

.. topic:: ``connectToWifi()``
   
   WiFi credentials are accessed here. Create your own ``credentials.h`` with ``#define SSID, PASS``.

.. topic:: ``WiFiEvent()`` 

   WIFIOUT_BIT_7 is set here when disconnection occurs to change behaviour of ``pollNewTask()``

.. topic:: ``onMqttConnect()`` 

   Hardcoded subscriptions can be placed here. Ideally in the future have the subscriptions read out of a modifyable variable/struct/JSON doc would
   allow for modification of those subscriptions. See Whomami implenetation in Roadmap to Further Development section. 

.. topic:: ``onMqttMessage()`` 
   
   MQTT payloads from subscriptions enter here. Payloads are read and operated on resulting in the possible setting or clearing of various EventBits: 
   Bits set: AUTH_BIT_1, ESTOPCLEAR_BIT_6, and ESTOPSET_BIT_6 can all be set within this function.
   Bits cleared: TIMEOUT_BIT_3, CARD_BIT_0, AUTH_BIT_1
   Task notification: 0'th bit of ``pollNewHandle`` is set to notify the ``pollNewTask()`` that a card has been denied and to blink red LEDs at the user.

.. warning::
   Care should be taken using ``strncmp()`` to evaluate topics and payloads. Specifically, the ``size_t num`` parameter should be utilized where possible as payloads may
   be coming from non-null terminating languages such as Java-script.


Proposed Future MQTT Topic Structure
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This is the proposed future MQTT Topic Structure. This structure would allow for much finer grained control at both the workshop and the individual tool level.
Changes to the code base will be necessary in order to make this viable. Primarily a means of telling each tool who it is and what workshop it is in so that
it could ammend more generic hardcoded MQTT topics. 

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


Roadmap to Further Development
-------------------------------

Optimization
^^^^^^^^^^^^^^

Whoami implementation for MQTT Topic Expansion
"""""""""""""""""""""""""""""""""""""""""""""""

As noted in the section on future MQTT topic structure each tool access system will need to be told what tool it is and what workshop it is in. This could be achieved 
using a JSON document payload, retained publishes with each devices factory coded MAC addresses, and a single hardcoded whoami topic.

Possible solutuion:

.. code-block:: C++
   :caption: ESP32 Generic Hardcoded MQTT Topics
 
   // These may have to live in a struct so that they may be editted
   tool/auth/req
   tool/auth/rsp
   tool/auth/eou
   
   // Subscriptions
   estop 
   tool/MAC // payload: JSON doc {tool: "alias", workshop: "woodshop" }

.. code-block:: C++
   :caption: Server side MQTT Topic

   // Retain flag set
   tool/MAC1  // payload: JSON doc {tool: "tablesaw", workshop: "woodshop" }
   tool/MAC2  // payload: JSON doc {tool: "bandsaw", workshop: "woodshop" }
   // etc

On boot ESP32 will recieved retained publish on /tool/myMAC with a JSON document as the payload. This will have to activate a helper task to unpack that JSON document
and ammend the /workshop/tool to all of the hardcoded MQTT topics. Task will have to unsubscribe from tool/myMAC on successful completion of topic editing lest the topic be repeatedly retriggered.

Requirements:

JSON implenetation for ESP32
MQTTammendTask design
Modificaiton of ``onMqttMessage()`` and ``onMqttConnect()`` to accept pointers through the ``void * pvTimerID`` in their ``xTimerCreate()`` call. This will be needed to give
scope to the callback functions so that MQTT topics can be read out of a struct or JSON doc or whatever implementation is decided upon.

.. code-block:: C++

   ESP.getEfuseMAC(); // This call will return the unique factory programed MAC address.


Interrupt functionality of the MFRC522 module
""""""""""""""""""""""""""""""""""""""""""""""

The MFRC522 chip supports interrupts generated on pin 5. The PCB design has left this pin unconnected so that is may be connected if desired. 

If this is to be pursued RTOS function calls will need to be changed to their ISR safe equivalents.

Optimizing RTOS Task Stack Size
"""""""""""""""""""""""""""""""""
Each RTOS task maintains its own stack and therefore on creation you must specify the depth of that stack. Determining how deep a stack to specify is somewhat of a guessing game,
fortunately RTOS makes some API calls available to help determine just how much stack any given task needs: ``usTaskGetStackHighWaterMark()``.

.. important::

   ESP32 RTOS specifies its stack depth in bytes! Not words! (Vanilla RTOS is the reverse of this).

The HighWaterMark API returns the minimum amount of **unused** stack of the stack depth allocated at task creation. It should be checked both at task creation and 
during task execution. Using this API the stack depth can be wittled down to its safest minimum.

.. code-block::
   :caption: Example usuage of the HighWaterMark API
   
   void exampleTask(void *params){
   UBaseType_t uxHighWaterMark;

   uxHighWaterMark = uxTaskGetStackHighWaterMark( NULL ); // Returns the minimum amount of unused stack space available since task creation
   // It is a good idea to check the high water mark at task creation (outside of the for(;;))

      for(;;){
         // and during the task execution
         uxHighWaterMark = uxTaskGetStackHighWaterMark( NULL ); // Returns the minimum amount of unused stack space available since task creation

         // depth of nested function calls can considerably increase the depth of stack you will need
      }
   }

All RTOS tasks created during my Summer co-op of 2020 have HighWatermark calls in place but commented out.  **None of my tasks have had their stack depth optimized.**

Other ways to know you've not specified enough stack? **Stack Overflow**
This usually manifests itself at the ESP32 going into a wild bootloop that is viewable on the serial monitor.

Shrinking program size for OTA
"""""""""""""""""""""""""""""""""""""

For the over the air updates functionality to be used our program must occupy <50% of flash memory. As of 2020/08/07 it occupies ~59%. Additionally as part of the OTA process logs from tools 
will have to be requested and transmitted before the OTA is initiated as this process will likely overwrite the SPIFFS partition.

This is most likely to be achieved by replacing as much of the Arduino C++ code with straight C code that utilizes the ESP32 native API.

Implementation of onboard logging
"""""""""""""""""""""""""""""""""""

In the event of an MQTT or WiFi disconnection event the toolAccess system grants unconditional access to whoever presents an RFID card to it. Ideally these UIDs would also be\
logged in internal flash memory for transmission to server upon reconnection. In design discussions it is presumed that this will likely be an uncommon event and therefore 
should not take a heavy toll on the limited read/write cycles of the ESP32 flash memory.

**How this should work**

1. A WiFi/MQTT disconnection event is detected
2. toolAccess system changes behaviour to grant unconditional access.
3. toolAccess system logs relative onboard time of disconnection event as first entry in a log file.
4. Every card that is granted access has its UID logged in the same file.
5. WiFi/MQTT connection re-established.
6. toolAccess system changes behaviour back to conditional access.
7. Last entry in log file is relative time of disconnection event.
8. Sever asks toolAccess system if it has logs.
9. toolAccess system responds with Y, size of logs/N
10. Server asks for logs.
11. toolAccess system transmits logs and deletes log file.

**Roadblocks**

1. SPIFFs has stream utility inheritances that during the Summer of 2020 I could not quite get my head around. However, this means that they will likely play ball with the Arduino
JSON library which would make writing and reading the log file much clearer at the cost of the log file being much larger.

2. Care would have to be taken to ensure that this functionality does not interefer with the OTA update requirement >50% of flash memory needing to be free.

Desired Future Features
"""""""""""""""""""""""""

1. Addition of other sensors



