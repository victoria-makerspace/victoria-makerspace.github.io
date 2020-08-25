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
3. `ESP-IDF FreeRTOS SMP Changes <https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/freertos-smp.html>`_ Espressif's documentation for the differences bewteen ESP32's RTOS and vanilla freeRTOS
4. `ESP32 FreeRTOS API Reference <https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/system/freertos.html>`_ The Espressif documentation for their flavour of FreeRTOS

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



**RTOS Task Summary**


.. +-------------------+---------------------+-------------------+-------------------+-------------------+
| Task Name         | Task Handle         | WaitBits()        | SetBits()         | 
+===================+=====================+===================+===================+
| pollNewTask()     | pollNewHandle       | N/A               | If(server) CARD_BIT_0
|                   |                     |                   | 

**RTOS Task List**

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

.. code-block:: C++
   :linenos:
   :caption: Polling for New Cards

   void pollNewTask (void *params){
      metaStruct *progParams = (metaStruct*) params;
      uint32_t notificationValue;

      for(;;){
         vTaskDelay(MS_POLL_TIMER_PERIOD); // Wait for at least he polling time

         notificationValue = ulTaskNotifyTake (pdTRUE, pdMS_TO_TICKS(300));
         if(notificationValue == 1 ){          // Notification from onMqttMessage that a card was denied
           Serial.println("Notification!!");
           Serial.println(notificationValue);
           // Set both LEDS to Red
           progParams->LEDParams0.myColour = CRGB::Red; 
           progParams->LEDParams1.myColour = CRGB::Red;
           // Set both LEDs to not blink
           progParams->LEDParams0.blink = 1;
           progParams->LEDParams1.blink = 1;
           vTaskResume(blinkLEDHandle0);
           vTaskResume(blinkLEDHandle1);
         }

         if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial ()){ // Poll for new cards BUT only when CARD_BIT_0 is not set

            mfrc522.PICC_HaltA(); // We have read the card now be halt it
            xEventGroupSetBits(rfidStatesGroup, CARD_BIT_0);                     // Now that we have detected a card set CARD_BIT_0 to unblock some Tasks

            // We need to check if the card's UID is authorized

            // First, check if to make sure the WiFi outtage flag hasn't been thrown
            if(xEventGroupGetBits(rfidStatesGroup) == 129){ // If the WiFi is out simply grant access and log

                // May still be useful for SPIFFS as storing a byte or int is more space efficient than a string. We can convert come re-connect to server
                userID(progParams, mfrc522.uid.uidByte, mfrc522.uid.size); // Move UID from struct to our own buffer
                Serial.println("WiFi is out");
                writeLog(progParams);
                xEventGroupSetBits(rfidStatesGroup, AUTH_BIT_1);  // Grant access unconditionally because WiFi is out
            }

         else{ // If the Wifi isn't out ask server for authorization
            progParams->card.uidStrLen = (mfrc522.uid.size*3); // Our string length will be 3x the length the equivalent byte value
            byteToHexStr(mfrc522.uid.uidByte, mfrc522.uid.size, progParams->card.uidStr, progParams->card.uidStrLen); // Now convert the byte in the mfrc522 struct to a string and dump it into our struct string     

            // Publish our uid string to find out if its authorized
            uint16_t packetIdPub1 = mqttClient.publish("rfid/auth/req", 1, false, progParams->card.uidStr); // Publish the read UID to rfid/auth/req. BE VERY CAREFUL TO NOT SET THE RETAIN FLAG
            Serial.print("Publishing at QoS 1, packetId: ");
            Serial.println(packetIdPub1);
          }
         // Now suspend our polling task until we hear back from MQTT broker OR if WiFi is out proceed with the request of the RFID control loop
         vTaskSuspend ( NULL ); // Suspend ourselves. No need to keep polling now that there is a card
       }

       else {
         // No new card, keep polling for new cards
         Serial.print("EventBits:");
        }
      }
   }


``pollNewTask()`` is the entry point for the RFID control loop. Once it detects a new card it suspends itself. It is very important it is resumed at the appropriate places for the control loop to keep
functioning (ie. whenever we return to state noCard, ``timeoutTask``). However, this also means that we can disable this task as a means of suspending the RFID functionality, see the EstopFire and 
EstopClear tasks for more details.

This task also checks for a task notification on line 7. This notification is made from ``onMqttMessage()`` when a card is denied by the server. The LEDParam structs are out of scope of the MQTT callback functions 
of the AsynMQTTClient library and I could not figure out a way of passing them the value. Instead ``pollNewTask()``, which is also resumed when a card is denied, is notified so that it may change the LEDParams
to blinking red.

When WiFi/MQTT is connect only CARD_BIT_0 is set in ``pollNewTask()`` and AUTH_BIT_1 is set elsewhere in ``onMqttMessgae()``. 
When a WiFi or MQTT disconnection event occurs WIFIOUT_BIT_7 is set. This is detected on line 9 and allows for both CARD_BIT_0 and AUTH_BIT_1 to be set within ``pollNewTask()`` without server authorization. 
This is where future implementation for flash memory logging should be placed.



.. closeRelayTask()
   waitBits = CARD_BIT_0 | AUTH_BIT_1
   setBits = RELAY_BIT_2
   vTaskSuspend ( NULL )

.. code-block:: C++
   :linenos:

   void closeRelayTask (void *params){
   
     metaStruct *progParams = (metaStruct*) params; // Should be static/const maybe?
   
     const TickType_t xTicksToWait = portMAX_DELAY; // Wait forever 
     const EventBits_t xBitsToWaitFor = (CARD_BIT_0 | AUTH_BIT_1); // This creates a mask for checking our bits. To make mask we always use |
     EventBits_t uxBits;                                          // But to check a mask we always use &
   
     for(;;){
       uxBits = xEventGroupWaitBits(rfidStatesGroup, xBitsToWaitFor, pdFALSE, pdTRUE, xTicksToWait);

       Serial.println("I'm going to close the relay!");

       // Set both LEDS to Green
       progParams->LEDParams0.myColour = CRGB::Green; 
       progParams->LEDParams1.myColour = CRGB::Green;
       // Set both LEDs to not blink
       progParams->LEDParams0.blink = 0;
       progParams->LEDParams1.blink = 0;

       Serial.println("Light the lights!");
       vTaskResume(blinkLEDHandle0); // Resume the LED task
       vTaskResume(blinkLEDHandle1); // Resume the LED task


       digitalWrite(RELAY_PIN, HIGH); // Close the relay

        // Set relayBit
       xEventGroupSetBits(rfidStatesGroup, RELAY_BIT_2);

       vTaskSuspend ( NULL ); // Suspend ourselves. No need to keep polling for new now that we know a card is there
      }
     }

``closeRelayTask`` unblocks once CARD_BIT_0 and AUTH_BIT_1 are set. It closes the relay, sets the RELAY_BIT_2 (not currently used for anything), changes the LEDs to a solid green to indicate this to the user.
The task then suspends itself once all of this has been achieved. This task is resumed in ``timeoutTask()``.

.. pollPresTask ()
   waitBits = CARD_BIT_0 | AUTH_BIT_1
   SemaphoreGive SPIMutexHandle
   vTaskDelay()
      if present
         Halt
      else
         clearBits = CARD_BIT_0 | AUTH_BIT_1
         setBits = TIMEOUT_BIT_3
         vTaskResume pollNewHandle
   SemaphoreTake SPIMutexHandle

.. code-block:: C++
   :linenos:

   void pollPresTask (void *params){
     //UBaseType_t uxHighWaterMark;
     EventBits_t uxBits;
     const TickType_t xTicksToWait = portMAX_DELAY; // Wait forever 
     const EventBits_t xBitsToWaitFor = (CARD_BIT_0 | AUTH_BIT_1 ); // This creates a mask for checking our bits. To make mask we always use |
     byte resultWake;
     byte bufferATQA[2];
     byte bufferSize = sizeof(bufferATQA);
   
     metaStruct *progParams = (metaStruct*) params; 

     for(;;){
       // Wait for CARD_BIT_0 and AUTH_BIT_1 to be set before proceeding
       uxBits = xEventGroupWaitBits(rfidStatesGroup, xBitsToWaitFor, pdFALSE, pdTRUE, xTicksToWait);
   
       xSemaphoreTake(SPIMutexHandle, portMAX_DELAY); // Must take the SPIMutex if you wish to proceed. This prevents presPoll and collPoll from trying to access SPI bus at same time
       
       vTaskDelay(MS_POLL_TIMER_PERIOD); // Only poll every 100ms?
   
       pollPres(progParams, &rfidStatesGroup, &blinkLEDHandle0, &blinkLEDHandle1, &pollNewHandle);
   
       xSemaphoreGive(SPIMutexHandle); // Now give up the Mutex handle so collPoll may execute
     }
   }

``pollPresTask``  unblocks when CARD_BIT_0 and AUTH_BIT_1 are set. Because ``pollPresTask``and ``collPollTask`` occur in the same state and both utilize the SPI bus a Mutex is given/taken by the two Tasks. 
If ``pollPres()`` detects that the authorized card has left the RF field the CARD_BIT_0 and AUTH_BIT_1 is cleared then the TIMEOUT_BIT_3 is set. This unblocks the ``timeoutTask()`` while placing both
``pollPresTask()`` and ``collPollTask()`` back in the blocked state. 

.. collPollTask ()
   waitBits = CARD_BIT_0 | AUTH_BIT_1
   SemaphoreTake SPIMutexHandle
      if coll 
         clearBits = CARD_BIT_0 |  AUTH_BIT_1
         setBits = TIMEOUT_BIT_3 | COLL_BIT_4
         vTaskResume pollNewHandle
      else
      // do nothing
   SemaphoreGive SPIMutexHandle

.. code-block:: C++
   :linenos:

   void collPollTask (void *params){ // Must sync with pollPres

     metaStruct *progParams = (metaStruct*) params;

     EventBits_t uxBits;
     const TickType_t xTicksToWait = portMAX_DELAY; // Wait forever 
     const EventBits_t xBitsToWaitFor = (CARD_BIT_0 | AUTH_BIT_1 ); // This creates a mask for checking our bits. To make mask we always use |
     char collCounter = 0;
   
     for(;;){

       uxBits = xEventGroupWaitBits(rfidStatesGroup, xBitsToWaitFor, pdFALSE, pdTRUE, xTicksToWait); // Block until card and auth bits are set

       xSemaphoreTake(SPIMutexHandle, portMAX_DELAY); // Must have the SPI mutex inorder to proceed.

       // Collision polling function call
       collPolling (progParams, &blinkLEDHandle0, &blinkLEDHandle1, &rfidStatesGroup, &pollNewHandle);

       xSemaphoreGive(SPIMutexHandle); // Now give up the Mutex handle so presPoll may execute
     }
   }

``collPollTask()`` unblocks when CARD_BIT_0 and AUTH_BIT_1 are set.


.. timeoutTask ()
   waitBits = TIMEOUT_BIT_3
   vTaskDelay MS_TIMEOUT_PERIOD
   waitBits = TIMEOUT_BIT_3 // two step block
   pub eou/to
   digitalWrite RelayOpen
   clearBits = TIMEOUT_BIT_3 | RELAY_BIT_2




.. Ignore for now
   eStopSetTask ()
   waitBits = ESTOPFIRE_BIT_5

.. eStopClearTask ()
   waitBits = ESTOPCLEAR_BIT_6


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
       progParams->LEDParams0.blink = 0;                // Set LED0 to continous one 
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

``connectToWifi()`` - WiFi credentials must be placed or accessed from here.

``WiFiEvent()`` - WIFIOUT_BIT_7 is set here when disconnection occurs

``onMqttConnect()`` - Hardcoded subscriptions can be placed here.

``onMqttMessage()`` - MQTT payloads from subscriptions enter here. Payloads must be read and operated on inside this function.

.. code-block:: 
   :linenos:
   
   void connectToWifi() {
      Serial.println("Connecting to Wi-Fi...");
      WiFi.begin(preferences.getString("ssid").c_str(), preferences.getString("password").c_str()); // Access WiFi creds stored in NVS
   }

Wifi credentials must be placed or accessed here. Accessing via the non-volatile storage currently shown is not practical for mass deployment as this would require manually caching it on all deployed units.
An alternative but similar locally cached system should be employed.


.. code-block:: C++
   
   void WiFiEvent(WiFiEvent_t event) {
    Serial.printf("[WiFi-event] event: %d\n", event);
    switch(event) {
    case SYSTEM_EVENT_STA_GOT_IP:
        Serial.println("WiFi connected");
        Serial.println("IP address: ");
        Serial.println(WiFi.localIP());
        connectToMqtt();
        // Clear the WiFi outage bit
        xEventGroupClearBits(rfidStatesGroup, WIFIOUT_BIT_7);
        break;
    case SYSTEM_EVENT_STA_DISCONNECTED:
        Serial.println("WiFi lost connection");
        xTimerStop(mqttReconnectTimer, 0); // ensure we don't reconnect to MQTT while reconnecting to Wi-Fi
		  xTimerStart(wifiReconnectTimer, 0); // Start wifiReconnectTimer immediately

        // Set WiFi outage bit to change our authorization scheme
        xEventGroupSetBits(rfidStatesGroup, WIFIOUT_BIT_7);
        break;
     }
   }

EventBits used to change behaviour based on WiFi connectivity must be toggled within this function. This is necessary so that in the event of a WiFi or server outtage the tools remain usable (ie. system 
defaults to granting access to anyone who presents a card) and to shift to logging those access requests in flash memory.

.. code-block:: C++
   :caption: onMqttConnect 

   void onMqttConnect(bool sessionPresent) {
      Serial.println("Connected to MQTT.");
      Serial.print("Session present: ");
      Serial.println(sessionPresent);

      // Sub to the rfid topic
      uint16_t packetIdSub2 = mqttClient.subscribe("rfid/auth/rsp", 2);
      Serial.print("Subscribing at QoS 2, packetId: ");
      Serial.println(packetIdSub2);
   
      // Sub to the estop topic
      uint16_t packetIdSub3 = mqttClient.subscribe("rfid/estop", 2);
      Serial.print("Subscribing at QoS 2, packetId: ");
      Serial.println(packetIdSub3);
   }

Hardcoded subscriptions to be made on boot should be placed in this function. However, subscriptions can made made elsewhere in code using same syntax.


.. code-block:: C++

   void onMqttMessage(char* topic, char* payload, AsyncMqttClientMessageProperties properties, size_t len, size_t index, size_t total) {
      Serial.println("Publish received.");
      Serial.print("  topic: ");
      Serial.println(topic);
      Serial.print("  qos: ");
      Serial.println(properties.qos);
      Serial.print("  dup: ");
      Serial.println(properties.dup);
      Serial.print("  retain: ");
      Serial.println(properties.retain);
      Serial.print("  len: ");
      Serial.println(len);
      Serial.print("  index: ");
      Serial.println(index);
      Serial.print("  total: ");
      Serial.println(total);
   
      int topicLength = (strlen(topic));
      int payloadLength = (strlen(payload));
      const char * rsp = "rfid/auth/rsp";
      const char * estop = "rfid/estop";

      // const char * subs[30];
      // sprintf(subs, "rfid/%s/sub", mqttClient.clientId)
      
      /* Possible pitfalls:
      Retained messages will fall straight through into this on boot!!!
      Be very careful how you use retains them.
      */

      // refid/auth
      // This should maybe be functionized for neatness
      if ((strncmp (topic, rsp, topicLength)) == 0){ // strncmp returns true if exact match
      
        Serial.println("We have a matching topic");
        
        if (((strncmp (payload, "auth", 4)) == 0)){ // Read only four indices just incase the payload is not null terminated
          Serial.println("authorized!");
          xEventGroupSetBits(rfidStatesGroup, AUTH_BIT_1); //Set the authorized bit
          xEventGroupClearBits(rfidStatesGroup, TIMEOUT_BIT_3); // Clear the timeout bit just in case we're in a timeout
        }
        else if(((strncmp (payload, "denied", 6)) == 0)){
          Serial.println("denied!");
          xEventGroupClearBits(rfidStatesGroup, (CARD_BIT_0|AUTH_BIT_1)); // Revoke card and authorization bit if an unauthorized card arrives. 
          vTaskResume(pollNewHandle); // Resume polling for new cards
          
        }
        else if(((strncmp (payload, "seekiosk", 8)) == 0)){
          Serial.println("seekiosk!");
          // Set kiosk bit?
        }
      }
      // rfid/estop
      else if ((strncmp (topic, estop, topicLength) == 0)){
        if (((strncmp (payload, "fire", 4)) == 0)){ // Read only four indices just incase the payload is not null terminated
          Serial.println("Fire the eStop!");
          xEventGroupSetBits(rfidStatesGroup, ESTOPFIRE_BIT_5); 
          
        }
        else if(((strncmp (payload, "clear", 5)) == 0)){
          Serial.println("Clear the eStop!");
          xEventGroupSetBits(rfidStatesGroup, ESTOPCLEAR_BIT_6); 
        }
      }
   }

onMqttMessage() is where published messages that our ESP32 is subsribed to arrive. This is also where EventBits that can only be set via server authorization are set (eg. AUTH_BIT_1,
ESTOPCLEAR_BIT_6, and ESTOPFIRE_BIT_5).


.. warning::
   Care should be taken using ``strncmp()``. Specifically, the ``size_t num`` parameter should be utilized where possible as payloads may be coming from non-null terminating languages such as Java-script.


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

As noted in the section on future MQTT topic structure each tool access system will need to be told what tool it is and what workshop it is in. At least if we are to
avoid having to hardcode each station.

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

.. code-block:: C++

   ESP.getEfuseMAC(); // This call will return the unique factory programed MAC address.


Interrupt functionality of the MFRC522 module
""""""""""""""""""""""""""""""""""""""""""""""

The MFRC522 chip supports interrupts generated on pin 5. The PCB design has left this pin unconnected so that is may be soldered to one of the ESP pins if desired. 

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
This often manifests itself at the ESP32 going into a wild bootloop. 

Shrinking program size for OTA
"""""""""""""""""""""""""""""""""""""

For the over the air updates functionality to be used our program must occupy <50% of flash memory. As of 2020/08/07 it occupies ~59%. Additionally as part of the OTA process logs from tools 
will have to be requested and transmitted before the OTA is initiated as this process will likely overwrite the SPIFFS partition.

Desired Future Features
"""""""""""""""""""""""""

1. Addition of other sensors

## Potential Pitfalls

