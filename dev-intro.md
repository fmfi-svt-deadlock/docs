Developer Intro
===============

Per-subproject considerations
-----------------------------

Make sure to read all READMEs in the repository that you are interested in.

Repositories
------------

The project has many components, and that means we have many repositories for them.

### Server and general

  - [`server`](https://github.com/fmfi-svt-deadlock/server). This repository hosts the source code for the server. It is also main repository for the documentation of the whole system. Design issues and issues not specific against any single component should be opened in this repository.

Other server-related repositories do not exist yet.

### Controller

  - [`controller-hw`](https://github.com/fmfi-svt-deadlock/controller-hw). This repo will contain schematics and PCB designs for the Controller.
  - `controller-sw`. Software for the Controller.
  - [`temp-controller.py`](https://github.com/fmfi-svt-deadlock/temp-controller.py). We needed to deploy 2 prototypes right away, so this is a temporary jerry-rigged controller software to be used on Raspberry Pi. Will be obsolete soon.
  - [`libmfrc522.py`](https://github.com/fmfi-svt-deadlock/libmfrc522.py). This is a library for controlling the MFRC522 module used in the Reader. It is the module that actually reads cards. It is used in temp-controller, because the deployed prototype is using an older concept of the reader, which cannot read cards on its own, it just mediates communication between the MFRC522 module and the Controller. In the new concept it will not be needed, and will be obsolete soon.

### Reader

  - [`reader-hw`](https://github.com/fmfi-svt-deadlock/reader-hw). Schematics and PCB design of the Reader.
  - [`reader-sw`](https://github.com/fmfi-svt-deadlock/reader-sw). Software for the Reader.
  - [`libreader.py`](https://github.com/fmfi-svt-deadlock/libreader.py). This is a Python library for interfacing with the Reader. It was made for the temporary controller, bud will be kept up-to-date as it allows almost any Python-capable system to use the Reader. It also allows for easier debugging and development of the Reader.

### Management interfaces

  - [`webui`](https://github.com/fmfi-svt-deadlock/webui). The web interface to Deadlock -- a console for managing controllers, points of access, and access rules.

### Utilities

  - [`hw-testing`](https://github.com/fmfi-svt-deadlock/hw-testing). This repo contains library for testing STM32 MCU powered embedded devices, like Reader and Controller. It also contains tests for these boards. New tests are written easily, and it is used in development of the system, as well as during the manufacturing for final checkout of the hardware.
