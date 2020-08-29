=======================
RFID Module - A Primer
=======================

Purpose of this document
-------------------------

The RFID module choosen for this project is sold as either "MFRC522 module" or "RC522 module" in both cases named for the "MFRC522" chip made by NPX Semiconductor.
The purpose of this document is to provide a primer on how the MFRC522 module functions. For some understanding how the module works may assist in their understand 
of the code written in the course of this project. The MFRC522 RFID chip can do much more than it is used for in this project. Therefore I will only focus 
on how the MFRC522 functions so far as it is relevant to this project. 

This most commonly utilized Arduino library utilized for the MFRC522 is a library written by `Miguel Balboa <https://github.com/miguelbalboa/rfid#development>`_ .
It has long since passed to community maintainers and has been frozen as of 2019.

There is also a `rewrite <https://github.com/makerspaceleiden/rfid>`_ of the library done by members of the Leiden Makerspace with somewhat extended functionality and
further documentation. 

I opted to use the Leiden Makerspace library.

Technical documentation:

1. `MFRC522 Datasheet <https://www.nxp.com/docs/en/data-sheet/MFRC522.pdf>`_
2. `MIFARE ISO/IEC 14443 PICC Selection <https://www.nxp.com/docs/en/application-note/AN10834.pdf>`_ 
3. `List of status codes and types <https://docu.byzance.cz/hardware-a-programovani/programovani-hw/knihovny/mfrc522>`_
4. `Mario Capurso's write up using MFRC522 Arduino library <https://diy.waziup.io/assets/src/sketch/libraries/MFRC522/doc/rfidmifare.pdf>`_

.. sidebar:: Glossary of MFRC522 Technical Abbreviations & Terms

    .. glossary::
       PICC
          Proximity Integrated Circuit Card

       PCD
          Proximity Coupling Device

       UID
          Unique IDentifier number set by factory on each PICC (bytes 0..6 of memory block 0) may be be 4-10 bytes.

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



Card detection
--------------------------------------

.. figure:: ./images/NPXPolling.png
   :align: center
   :width: 50%

Card polling block diagram from `MIFARE ISO/IEC 14443 PICC Selection <https://www.nxp.com/docs/en/application-note/AN10834.pdf>`_ (Pg 5).

The MFRC522 polls for cards according to the diagram above. The RF field is switched on, if a card is present the RF field will provide power to the PICC which boots 
into an *Idle* state. The MFRC522 then sends out a REQA/B command (A/B are two different variants of the same command) which invites a PICC to enter state *Active*.
If a PICC is present it will enter state *Active* and it will respond to the REQA with an ATQA. 

.. important:: 
   Only *Active* or *Idle* PICC's will respond to an ATQA. The functions ``mfrc522.PICC_IsNewCardPresent()`` and ``mfrc522.PICC_RequesA()`` send out this REQA. 
   State *Halt* will not responded to an REQA but must first be invited to state *Active* via a WUPA which is executed but the function ``mfrc522.PICC_WakeupA()`` 

Upon receipt of a STATUS_OKAY from an ATQA the MFRC522 may then proceed to activate that card and preform a transaction. For our purposes that transaction is the extraction 
of the UID which occurs during card selection. Card selection is carried out by ``mfrc522.PICC_ReadCardSerial()`` which if successful places the extracted UID in it's class variable
``mfrc522.uid.uidByte`` along with ``mfrc522.uid.size``. 

Both detection of a new card and reading of that cards uid can be preformed with a single conditional card to form a polling routine.

.. code-block:: C++
   :caption: Card polling routine

   // Some means of non-blocking delay here
   if(mfrc522.PICC_IsNewCardPresent() && mfrc522.ReadCardSerial(){
      // mfrc522.PICC_IsNewCardPresent() - Returns 1 if StatusCode result == STATUS_OKAY || STATUS_COLLISION
      // mfrc522.ReadCardSerial() - Returns 1 if that card's uid can be read


      // Execute code to initiate state transition

      // Extract uid from mfrc522 class variable
      mfrc522.PICC_HaltA(); // This call must be must be made to prevent repeated re-detection of our card 
   }

Card presence
--------------

The minimum viable product for this project calls for (in most use cases) the end user to leave their access card in contact with the tool access system for access to that tool to persist. This necessitates 
that our system has some means of detecting the continued presence of an already authorized card. This functionality is already implied in the control loop described above. 

Only *Active* or *Idle* cards will respond to an REQA command and *Halted* cards will not.
Only *Halted* cards will respond to a WUPA command and *Active* or *Idle* cards will not.

This provides a very simple cards persistance polling routine where we simply need to execute a WUPA command. 

.. code-block:: C++
   :caption: Card Presence Polling Routine

   void pollPres(){

   // Declare buffer for the STATUS_CODE returned in the ATQA buffer
   byte bufferATQA[2];
   byte bufferSize = sizeof(bufferATQA);


   // Reset baud rates
	mfrc522.PCD_WriteRegister(mfrc522.TxModeReg, 0x00);
	mfrc522.PCD_WriteRegister(mfrc522.RxModeReg, 0x00);
	// Reset ModWidthReg
	mfrc522.PCD_WriteRegister(mfrc522.ModWidthReg, 0x26);
      
      if(mfrc522.PICC_WakeupA(bufferATQA, &bufferSize)){

         // A halted card has responded to WUPA therefore our card is still there

         mfrc522.PICC_HaltA(); // We must re-halt it now that we've woken it up
      }
      else{
         // No response to WUPA our card must have left
         // Our state has changed to noCard
         // Execute code to change state
      }
   }

The above example is not a 1:1 representation of the ``pollPres()`` function found in ``tollAccessRTOS.cpp``. Code from initiation state changes and interacting with the LEDs have been omitted for clarity.

Collisions
------------

But what if there is more than one card present in the RF field (a collision)? The MFRC522 can detect if a collision has occurred and has an anti-collision process to ensure only one card is read at a time.
Other colliding cards must wait for subsequent REQA commands to be read. The MFRC522 also throws an error code to indicate that a collision has occurred. This error code (``STATUS_COLLISION``) should be 
returned in the ATQA buffer received by our ESP32 as a result of the ``mfrc522.PICC_IsNewCardPresent()`` call used in our polling routine.

.. important:: 
   
   In practice this does not occur. In the event of a collision ``mfrc522.PICC_IsNewCardPresent()`` returns a 0 because it received a ``STATUS_TIMEOUT``. This behaviour also occurs in response to the 
   ``PICC_WakeupA()``.
   
This is a major problem because collisions result in the return of the same status code (``STATUS_TIMEOUT``) that we expect when our card is removed. Additionally it can create false negatives
for detection of new cards and doesn't leave us with a reliable means of detecting collisions.

This behaviour is noted both the commenting of MiguelBalBoa's library and in Mario Capurso's write up. In both cases poor antenna design is purposed as the case however it is beyond my capabilities 
to lend credence this hypothesis however the MFRC522 modules themselves do appear to have a QC issue with 15 out of 35 of those ordered initially for this project being unable to communicate with 
my test setup.

Once again the solution to our issue is hidden in the datasheet only this time it is unclear to the author if this is a deliberate feature or merely a byproduct of the protocol that simply happens to 
be a serviceable means of detecting a collision. 

The anticollision routine carried out by the MFRC522 ensures that one card is read per REQA command. The colliding card/cards must wait for addition REQA commands to be sent before they can be read.
This means that we can poll for a collision simply by calling a ``mfrc522.PICC_IsNewCardPresent()`` (recall that this function calls REQA) repeatedly in a loop and counting the 1'a returned by it. 

.. code-block:: C++
   :caption: Collision Polling detection

   void collPolling(){ 
      for (int i = 0; i < 4; i++){ 
         
         if (mfrc522.PICC_IsNewCardPresent()){ // IF PICC_IsNewCardPresent returns 1 > once we have a collision or a misaligned card
         //inc counter
         (progParams->card.collCounter)++;
        }
        else{
         // Do nothing
        }
      }
      if (progParams->card.collCounter >= 2){
         // We have a collision
         // Unconditional GOTO state TIMEOUT
      }
      else{ // No collision
         // No collision 

         
         //xEventGroupClearBits(*rfidStatesGroup, COLL_BIT_4); // We can revoke the collision flag if it was previously set because the offending card has been removed
      }
      progParams->card.collCounter = 0; // Reset the collision counter
      }
      
.. note::
   This method reliably detects collisions however it will also produce false positives if a single card is misaligned on the reader. Given that there is to be a cardholder that fixes the alignment
   of the card to the reader is this not likely to be a problem.

As can be seen in the code above if a collision is detected we unconditionally go to state timeout. This is because although the above polling routine can detect a collision it cannot reliably determine
how a collision is resolved, ie. which card was removed. The simplest resolution to this problem is simply to go to a timeout state which can be interrupted by re-authorizing one of the cards or allowing
the timeout to occur.