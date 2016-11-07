Design of modular protocol stack for Desfire cards
==================================================

What is a Desfire card?
-----------------------

Let's think of it as of a contactless memory card with access control.

According to [this](http://www.nxp.com/products/identification-and-security/mifare-ics/mifare-desfire/mifare-desfire-ev1-contactless-multi-application-ic:MIFARE_DESFIRE_EV1_2K), DESFire card is a `ISO/IEC 14443A` compatible card which uses optional `ISO/IEC 7816-4` commands. It can store several Applications with different access permissions, with several Files in each Application.

What protocols are in play?
---------------------------

So, Reader must be equipped with `ISO/IEC 14443A` (part 1 and 2) compliant Reader (reffered to as PCD in `ISO/IEC 14443`). In our case, this is a `MFRC522` module connected over SPI. Driver for this module must be implemented in software.

`ISO 14443A` parts 3 and 4 must be implemented in the software in order to perform anticollision loop, select a card and activate it. According to the standard, a card is activated by performing an 'Anticollision' loop. This allows selection of a particular card (called PICC) from several cards which may be in the RF field generated by the PCD.

DESFire then has it's own set of commands. These can be sent either wrapped in `ISO/IEC 14443-4` data blocks or wrapped in `ISO/IEC 7816-4` APDUs. In addition, DESFire card supports several `ISO/IEC 7816-4` commands.

What do we need to do with the card?
------------------------------------

The first iteration needs only to read UID of the card. Therefore everything above `ISO/IEC 14443-3` is unnecessary. However, it is almost certain that in the future we will want to do something more, so the system should be designed to allow easy addition of `ISO/IEC 14443-4`, `ISO/IEC 7816-4` and DESFire protocol library (on top / using encapsulation defined by either of these libraries).

First iteration is using `MFRC522` module connected over SPI. It is quite probable, however, that in the future the module may be changed for other type, or the same type can be kept but may be connected over some other interface (`MFRC522` supports SPI, I2C and UART).

This implies nice modular design. If done properly, parts of this stack may be reused in some other future projects.

Existing designs
----------------

The obvious thing to consider is to use some existing opensource implementation of this stack. Unfortunately most libraries strictly adhere to design principle "fuck modularity, everyone will be using just one MFRC522 over SPI with Arduino reading only UIDs" and implement all the afromentioned funcionality in one file of spaghetti code. Unfortunately such implementations are unusable for us.

Card stack design
-----------------

I've tried to take the most promising-looking existing implementation and modularize it, but found out that it would be easier to write it from scratch.

It was previously described that we need a modular design since we may be changing components. However, if we overdo it with modularity we will end up with hard-to-use hard-to-maintain code (e.g. everything-independent `MFRC522` library then requires design of platform-specific SPI / USART / I2C bindings and everything). A sane compromise must be found.

We are using and will continue using ChibiOS. The whole Reader application will use ChibiOS's facilities, will be multithreaded and this card stack should be designed with this in mind. It will have to integrate nicely with existing HAL of ChibiOS (which is by itself really nicely porable and can be used independently of ChibiOS/RTOS, since OSAL (Operating System Abstraction Layer) is placed between the RTOS and HAL).

The stack shlould look like this (top to bottom):

  - DESFire Protocol Library
  - (optional) `ISO/IEC 7816-4` Protocol Library
  - `ISO/IEC 14443-3` and `ISO/IEC 14443-4` Protocol Libraries
  - `MFRC522` driver

The stack must support swapping of implementation of various layers and using several different implementation of layers simultaneously (for example, the device could have 2 different RFID modules connected, this library must allow these two to be used simultaneously). Nice modular separation will also allow for mocking various modules for testing purposes.

This modularity will be achieved by object-oriented design. Each lower layer will create a structure with pointers to functions of the given lower layer for the upper layer to use. This structure will have fixed common API and a single `void` pointer to a lower-layer specific structure containing data the lower layer needs. Upper layer will then call these functions from the object with the object itself as a first parameter. This will allow encapsulation and usage of several objects from different layers simultaneously. The upper layer may, if necessary, wrap this object to a object of it's own, to add custom data for working with this object and for exporting this object to even upper layers.

### DESFire Protocol Library and `ISO/IEC 7816-4` Protocol Library

These are not needed at this point. However it is guaranteed that they can be written easily using `Card` objects provided by the `ISO/IEC 14443` Protocol Library.

### `ISO/IEC 14443` Protocol Library

This library implements `ISO/IEC 14443` part 3 (Initialization and Anticollision) and part 4 (Transmission Protocol). It operates on top of an abstract `ISO/IEC 14443` PCD API, which the PCD driver implements (described bellow).

`ISO/IEC 14443` defines two types of cards: A and B. Only part A will be implemented, because we don't need the part B right now (we have no hardware capable of communicating with B-type cards).
The library will however be written with part B of the standard in mind so that it can be easilly added later if needed. Names of functions related to A part of the standard will end with `A`, B-function names will end with `B` respectively, and common function names will end with `AB`.

Although the library itself tries to be stateless, Reader is a state machine and Card is a state machine, which respons differently to the same stimuli when in different state. State of these entities is therefore encapsulated in structures `Pcd` and `Card`. All functions contain state checks on these objects to prevent undesired behaviour. During initialization both `Pcd` and `Card` structures must be passed as parameters to functions, after the card is activated it is bound to a `Pcd` (until deactivation) and `Card` structure contains pointer to the `Pcd` structure, so only `Card` structure has to be used.

`ISO/IEC 14443A` defines a way to detect and communicate with multiple cards in the same RF field. All `ISO/IEC 14443A` cards must support mechanism for detecting multiple cards, however, mechanism for activating and communicating with several cards simultaneously is optional. If PCD wants to activate several cards at once the workflow is as follows:

  - Anticollision loop runs, and when it finishes the PCD knows `UID`s of all PICCs in the RF field.
  - PCD can then request activation of some PICC based on it's `UID`, and it can assign it a `CID`. `CID` is a unique identifier (between 1-15 inclusive) of a PICC in the RF field.
  - PICC then tells the PCD whether it supports the `CID` feature.
    + If PICC does support `CID` the PCD sends blocks containing this `CID` to this PICC and will not use this `CID` with other PICC while the current PICC is active. It also can't activate any PICC which does not support `CID`.
    + If PICC does *not* support `CID` the PCD will send blocks without `CID` and can't activate any other PICC while the current PICC is active.

The library will therefore obey these rules and:

  - If there is no active card it will allow activation of any card.
  - If there is an active card which doesn't support `CID` it will not allow activation of any other card.
  - If there is an active card which supports `CID` it will allow activation of other card which supports `CID` (but no more than 15 cards in total).
  - If there is an active card which supports `CID` it will not allow activation of other card which does not support `CID`.

The library will handle assigning `CID`s internally. It will keep state in the `Pcd` structure.

When the card is selected optionally communication speed parameters may be negotiated. Request for negotiation, however, must be the first thing a PICC receives (`CID` is taken into account at this stage), because if it receives any other valid data block it will automatically disable the negotiation feature. Support of this feature is also optional.
This library contains `select` functions which will select a PICC and assign a `CID` (if applicable). After this function is used on a card either `set_parameters`, `set_best_parameters` or `set_default_parameters` functions may be used. These will set the communication parameters for the card and remember them in the `Card` structure (so that everytime we communicate with a PICC the PCD can be reconfigured accordingly). After these functions are used normal communication with the PICC may start.

`ISO/IEC 14443-4` defines a half-duplex request-response communication protocol. When a PCD sends a request it must wait either for the response or for the timeout to occur. It can't transmit anything to the card during this time. Although not forbidden by the standard, this library will also disallow sending two requests to two different cards (with different `CID`s) at the same time, as when this happens there is a risk that the response of the first card will be lost. Not to mention the possible necessity of switching communication speeds for different `CID`s. This library therefore exports function `send_receiveAB`. Use of this function is strongly encouraged as it simplifies the development. This function will block until either the response is received or an error (such as a timeout) occurs. Internally, this function uses semaphores for waiting, and therefore allows other threads to run (it does not busy-wait if underlying OSAL supports it).

For flexibility this library also provides functions `send` and `receive`. These functions return immediatelly and have appropriate locking in place so that `send` cannot be used when some other communication is in progress or `receive` has not yet been called. Using these complicates application code, so their use is discouraged.

#### API

  - `get_cards_in_fieldA(Pcd, Card*)`: This function performs an anticollision loop as defined in `ISO/IEC 14443A` and returns an array of `Card`s. The `Card` structure will contain an UID of the card. By design, after successfully performing an anticollision on a card the card transitions to an `ACTIVE` state, and that is a side-effect we don't really expect of this function, therefore after activating new cards it also `HALT`s them. This function also allocates new `Card` objects. If you don't use them, don't forget to `free_cardAB` them.
  - `construct_cardA(uid, uid_length)`: This function constructs a new `Card` structure.
  - `select_cardA(Pcd, Card*)`: This function attempts to select a card. The Card structure may come either from `get_cards_in_fieldA` or may be constructed by user (useful if expected card UID is known in advance). Only `uid` and `uid_length` fields of the `Card` structure are taken into account, this function will properly initialize all other fields. This function will also obey all rules related to `CID` described above.
  - `detect_and_select_single_cardA(Pcd, Card*)`: This function performs an anticollision loop as defined in `ISO/IEC 14443A` and activates card with the highest `UID` it finds. This function will not provide any further info about other cards in the field, however, it is a bit faster than using `get_cards_in_fieldA` and `activate_cardA`. This function is useful if it is reasonable to assume that only one card will enter the RF field at a time.
  - `deactivate_cardAB(Pcd, Card*)`: This function will deselect previously selected card, freeing resources associated with this card and putting the card to a `HALT` state.
  - `set_parametersA(Card*, transmit_speed, receive_speed)`: This function will configure the PICC to use these communication parameters. It will fail if either PICC or PCD doesn't support these parameters, or if the PICC doesn't support setting communication parameters in general.
  - `set_best_parametersA(Card*)`: This function will set the fastest communication parameters supported by both the PICC and the PCD. If PICC does not support setting parameters in general this function succeeds and leaves the parameters at their default values (because, semantically, that is the best the card can do).
  - `set_default_parametersA(Card*)`: This function will skip the parameters negotiation entirely and will just use the defaults.
  - `send_receiveAB(Card*, buffer*, length)`: This function will send data to the PICC, wait for the response and return the response in the `buffer`. If the `buffer` is not big enough to hold the whole response only part of the response which fits the buffer will be returned and the rest can be obtained using `get_responseAB(Card*, buffer*, length)`. This function will block until the whole response is received.
  - `sendAB(Card*, buffer*, length)`: This function sends data to the PICC. It will return immediatelly.
  - `receiveAB(Card*, buffer*, length)`: This function will return immediatelly either the received response or error that the response has not yet been received.
  - `receive_waitAB(Card*, buffer*, length)`: This function will return the received response and will block until the response is received.
  - `free_cardAB(Card*)`: This functions `frees` the `Card` structure and all resources associated with it. Make sure it is not used afterwards.

Minor state-inquiring and plumbing functions are ommited for clarity.

#### Exports to other layers

This library provides standardised object `Card` for use with other layers. The `Card` object is a structure which contains pointers to functions `send_receiveAB`, `sendAB`, `receiveAB` and `receive_waitAB` and one `void` pointer which other layers shouldn't touch.

The purpose of the last `void` pointer is to point at the internal structure used by this library to keep state of the card and other info about the card which other layers should not have to know about. This allows for some other library to provide its own `Card` structures with completely different internal representation of state and everything.

It goes without saying that functions of this library are designed to work only on `Card` structures created by this library. If you pass `Card` structure created by other library as a parameter you will get what you deserve: undefined behaviour.

#### Memory management

This library wont `malloc` nor `free` any `Pcd` structures.

This library **caries sole responsibility** for `malloc`ating and `free`ing its `Card` structures. User **shall not** `malloc` his own `Card`s nor `free` any `Card`s allocated by this library. For each `Card` allocated by this library also internal structure for internal data is allocated. If the user would `free` the `Card` reference to this internal structure would be lost, resulting in a memory leak. Functions `construct_cardA(uid, uid_length)` and `free_cardAB(Card*)` should be used instead.

However, it is responsibility of the **user** to call `free_cardAB(Card*)` on `Card` structures he no longer wishes to use.

It is also responsibility of the **user** to `malloc` and `free` data buffers passed to various send and receive functions. This library can be compiled with `SAFE_BUFFERS` option, which will cause that whenever a function for sending data is called the data buffer is copied to the internal buffer before the function returns. If this library is compiled without the `SAFE_BUFFERS` option the passed buffer is used directly. This can save memory, but then **the user has to ensure that the transmit buffer is not modified nor freed until the response is received**.

#### Threading and thread safety

This library is designed to be used in a multithreaded enviroment. It won't create any threads for itself, however all functions are thread-safe (if underlying OSAL correctly implements locking primitives), and those that semantically cannot be used concurrently will return appropriate errors.

#### States and transitions

##### States and transitions of a `Card`

Function can be used only if it is indicated on transition edge here, otherwise the function will fail.

Functions `deactivate_cardAB` and `free_cardAB` were ommited for clarity. These can be used in any state (except `BUSY`) and will set the state to `IDLE` or `NONEXISTENT`, respectively.

```eval_rst

.. graphviz::

    digraph card_state_transitions {
        NONEXISTENT      -> IDLE             [label="get_cards_in_fieldA()\n construct_cardA()"]
        IDLE             -> SET_PARAMS       [label="select_cardA()"]
        NONEXISTENT      -> SET_PARAMS       [label="detect_and_select_single_cardA()"]
        SET_PARAMS       -> READY            [label="set_params()\n set_best_params()\n set_default_params()"]
        READY            -> BUSY             [label="send_receiveAB()\n sendAB()"]
        BUSY             -> PENDING_RESPONSE [label="response received"]
        PENDING_RESPONSE -> READY            [label="send_receiveAB\n receive_waitAB\n returns whole message"]
        PENDING_RESPONSE -> READY            [label="receiveAB\n returns whole message"]
        PENDING_RESPONSE -> PENDING_RESPONSE [label="send_receiveAB\n receive_waitAB\n receiveAB\n returns partial message"]
    }

```

#### Tests

`ISO/IEC 14443` part 4 defines several example communication scenarios which could be used when designing compliance tests. However, implementing those tests is still an open problem.

Although in tests the `Pcd` can be (by the virtue of modular design) easily mocked the same cannot be said about the OSAL (abstraction on top of the ChibiOS/RT) which this library relies upon. Research will be done in this area.

### `MFRC522` driver

This layer presents upper layers with an abstract `ISO/IEC 14443` PCD API and implements drivers for the MFRC522 module. Since the MFRC522 chip can be connected through multiple interfaces (SPI, USART, I2C) this driver itself must be modular. The lower modules handles adressing, reading and writing registers of the MFRC522 over various interfaces. The upper module, which uses abstract `read_register` and `write_register` API handles the business logic of the driver.

#### Abstract API

The biggest design challenge for this driver is the abstract API. This API has to be abstract enough so that drivers for various readers implementing it can be written, however, it should also be easy to comprehend, should adhere to the `ISO/IEC 14443-3` standard and, ideally, easy to implement.

TODO.