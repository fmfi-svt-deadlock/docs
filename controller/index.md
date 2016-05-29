Controller
==========

This is a (usually embedded) device with network connectivity to the server. Its job is to open and close single door it controls when it receives information about a card from the Reader. It also tells Server about everything that is happening with the door (like usage of a manual lock override (opening doors with a physical key) or manual door override (like kicking the door out)). It can be connected to up to 2 readers (possibly more with a proper extension board).

This device will be gradually developed, with 2 models. First model (Controller Model A) will be able to function only if it has an active connection to the server, later more advances model (Controller Model B) will be able to perform it's decisions offline if the server is down.

It will also be extensible with 'Controller Extension Boards', that can provide extended functionality.
