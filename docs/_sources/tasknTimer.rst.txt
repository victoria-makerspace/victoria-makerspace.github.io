================================
RTOS Tasks and Timers Reference
================================

RFID Tasks
------------------

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

EStop Tasks
--------------

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


Relay Task
-------------

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


Addressable LED Tasks
----------------------

Library \: `FastLED <https://github.com/FastLED/FastLED>`_

Control of the COM12999 addressable LEDs is done via the FastLED library, a single RTOS task definition ``blinkyLEDTask`` and a single software timer ``LEDblinkTimer``.

.. code-block:: C++
   :caption: blinky LED Task
   :linenos:

   void blinkyLED (void *params){

      LEDParams *l = (LEDParams*)params; // Dumping our struct parameters into task instance of LEDParams via casting of void *params to LEDParams

  
      for (;;){ // Infinite loop required for RTOS tasks b/c if allowed to return they would delete

         // Branch based on blink status  
         if(l->blink){                             // if blink flag has been set

            if (l->blinkTemp){                 // If the blinkTemp flag is set we start the LEDblinkTimer
                 l->blinkTemp=0;               // Only want the timer started once
                xTimerStart(LEDblinkTimer, 0); // Start the timer
            }

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

    void LEDTimerCallback (TimerHandle_t LEDblinkTimer){
        metaStruct *progParams = (metaStruct*) pvTimerGetTimerID (LEDblinkTimer);       

        progParams->LEDParams0.myColour = CRGB::Black; // Once our timer expires our callback function simply resets out LEDs to off
        progParams->LEDParams1.myColour = CRGB::Black;
        progParams->LEDParams0.blink = 0;
        progParams->LEDParams1.blink = 0;
    }

Where the LED control tasks differ slightly from the other RTOS tasks is at creation they are passed the address of their parameter struct rather than the address of the metaStruct. The LED tasks
need only access to the LED parameters while other tasks needs access to their own parameters and the LED parameters.

.. code-block:: C++
   :caption: LED Task and Timer Creation in void setup ()

   //Task creation
   xTaskCreatePinnedToCore(blinkyLED, "blinkLED", 1024, &progParams.LEDParams0, 1, &blinkLEDHandle0, 1);
   xTaskCreatePinnedToCore(blinkyLED, "blinkLED1", 1024, &progParams.LEDParams1, 1, &blinkLEDHandle1, 1);

   // Software timer creation
   LEDblinkTimer = xTimerCreate("LEDblinkTimer", LEDBLINK_PERIOD, pdFALSE, (void*)&progParams, reinterpret_cast<TimerCallbackFunction_t>(LEDTimerCallback));
   
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


MQTT software timers and callback functions
---------------------------------------------

The MQTT funcitonality utilizes software timers to handle disconnection/reconnection events from WiFi and the MQTT server. These are found in ``void setup()`` of
``toolAccessRTOS.ino``. 

.. code-block:: C++

    mqttReconnectTimer = xTimerCreate("mqttTimer", MS_MQTT_RECONNECT_PERIOD, pdFALSE, (void*)0, reinterpret_cast<TimerCallbackFunction_t>(connectToMqtt));
    wifiReconnectTimer = xTimerCreate("wifiTimer", MS_WIFI_RECONNECT_PERIOD, pdFALSE, (void*)0, reinterpret_cast<TimerCallbackFunction_t>(connectToWifi));

As mentioned above, the MQTT functions are those included in the ``FullyFeatured-ESP32.ino`` example included with the library. Most of them remain unmodified, the exceptions to this are
``connectToWifi()``,  ``WiFiEvent()``, ``onMqttConnect()``, and ``onMqttMessage()`` . As such I will only discuss these functions here.

.. topic:: ``connectToWifi()``
   
   WiFi credentials are accessed here. Create your own ``credentials.h`` with ``#define SSID, PASS``.

.. topic:: ``WiFiEvent()`` 

   WIFIOUT_BIT_7 is set here when disconnection occurs to change behaviour of ``pollNewTask()``

.. topic:: ``onMqttConnect()`` 

   Hardcoded subscriptions can be placed here. Ideally in the future have the subscriptions read out of a modifyable variable/struct/JSON doc would
   allow for modification of those subscriptions. See Whomami implementation in Roadmap to Further Development section. 

.. topic:: ``onMqttMessage()`` 
   
   MQTT payloads from subscriptions enter here. Payloads are read and operated on resulting in the possible setting or clearing of various EventBits: 

   Task notification: 0'th bit of ``pollNewHandle`` is set to notify the ``pollNewTask()`` that a card has been denied and to blink red LEDs at the user.

   +-------------------------+------------+------------------+-------------------------+---------------------------------+-----------------+
   | Topic                   | Payload    |  SetBits()       | ClearBits()             | xTaskNotify()                   | vTaskResume ()  |
   +=========================+============+==================+=========================+=================================+=================+
   | rfid/auth/rsp           | auth       | AUTH_BIT_1       |  N/A                    | N/A                             | N/A             |                    
   +                         +------------+------------------+-------------------------+---------------------------------+-----------------+                
   |                         | denied     | N/A              | CARD_BIT_0 | AUTH_BIT_1 | pollNewHandle, (1<<0), eSetBits | pollNewHandle   |
   +                         +------------+------------------+-------------------------+---------------------------------+-----------------+
   |                         | seekiosk   | N/A              | N/A                     | N/A                             | N/A             |
   +-------------------------+------------+------------------+-------------------------+---------------------------------+-----------------+
   | rfid/estop              | fire       | ESTOPFIRE_BIT_5  | N/A                     | N/A                             | N/A             |
   +                         +------------+------------------+-------------------------+---------------------------------+-----------------+
   |                         | clear      | ESTOPCLEAR_BIT_5 | N/A                     | N/A                             | N/A             |
   +-------------------------+------------+------------------+-------------------------+---------------------------------+-----------------+

   .. warning::
      Care should be taken using ``strncmp()`` to evaluate topics and payloads. Specifically, the ``size_t num`` parameter should be utilized where possible as payloads may
      be coming from non-null terminating languages such as Java-script.
