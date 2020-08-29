===============================
Roadmap to Further Development
===============================

Proposed Future MQTT Topic Structure
--------------------------------------

The current MQTT topic structure used for the prototype is quite simple for testing purposes. It will have to be expanded and complicated once >1 device is to be controlled. 
This is the proposed future MQTT Topic Structure. This structure would allow for finer grained control at both the workshop and the individual tool level.
Changes to the code base will be necessary in order to make this viable. Primarily a means of telling each tool who it is and what workshop it is in so that
it could amended more generic hardcoded MQTT topics. 

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

Whoami implementation for MQTT Topic Expansion
-----------------------------------------------

As noted in the section on future MQTT topic structure each tool access system will need to be told what tool it is and what workshop it is in. This could be achieved 
using a JSON document payload, retained publishes with each devices factory coded MAC addresses, and a single hardcoded whoami topic.

Possible solution:

.. code-block:: C++
   :caption: ESP32 Generic Hardcoded MQTT Topics
 
   // These may have to live in a struct so that they may be edited
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
----------------------------------------------

The MFRC522 chip supports interrupts generated on pin 5. The PCB design has left this pin unconnected so that is may be connected if desired. 

If this is to be pursued RTOS function calls will need to be changed to their ISR safe equivalents.

Optimizing RTOS Task Stack Size
---------------------------------
Each RTOS task maintains its own stack and therefore on creation you must specify the depth of that stack. Determining how deep a stack to specify is somewhat of a guessing game,
fortunately RTOS makes some API calls available to help determine just how much stack any given task needs: ``usTaskGetStackHighWaterMark()``.

.. important::

   ESP32 RTOS specifies its stack depth in bytes! Not words! (Vanilla RTOS is the reverse of this).

The HighWaterMark API returns the minimum amount of **unused** stack of the stack depth allocated at task creation. It should be checked both at task creation and 
during task execution. Using this API the stack depth can be wittled down to its safest minimum.

.. code-block:: C++
   :caption: Example usage of the HighWaterMark API
   
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
---------------------------------

For the over the air updates functionality to be used our program must occupy <50% of flash memory. As of 2020/08/07 it occupies ~59%. Additionally as part of the OTA process logs from tools 
will have to be requested and transmitted before the OTA is initiated as this process will likely overwrite the SPIFFS partition.

This is most likely to be achieved by replacing as much of the Arduino C++ code with straight C code that utilizes the ESP32 native API.

Implementation of onboard logging
----------------------------------

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


2. Care would have to be taken to ensure that this functionality does not interfere with the OTA update requirement >50% of flash memory needing to be free.

Desired Future Features
--------------------------

1. Addition of other sensors
