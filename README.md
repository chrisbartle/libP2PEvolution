# libP2PEvolution
Library to facilitate peer-to-peer between applications across the Internet

This library is intended to be an evolution of previous P2P (peer to peer) approaches. The evolution comes from combining established technologies. Specifically:
- All communication will be done with AES
- DHT (distributed hash tables) will be used to facilitate matchmaking across the Internet
- UDP broadcasts will facilitate matchmaking locally
- All communication will be done via UDP. TCP emulation will be done on top of UDP as well as other communication types.
- Rate limiting can be done both through packet loss (as with traditional TCP) but also be measuring packet transit times (as with LEDBAT)

# General Design

libP2PEvolution is a c++ library. It is intended to build as cleanly as possible and will attempt to limit 3rd party dependencies. Network communication will be based on the Asio library.

## libP2PEvolution class

The core libP2PEvolution class must be instantiated in order to use this peer to peer functionality. Each instance of this class must be provided with a meeting ID. The meeting ID is central to the design of this library as it is used for everything:
- The meeting ID is a standard ASCII string that is provided by the calling code
- The meeting ID acts as the AES encryption key for all peer-to-peer communication
- A hash of the meeting ID is used by DHT to locate other clients in the same meeting
- The local UDP port number is derived from the meeting ID

