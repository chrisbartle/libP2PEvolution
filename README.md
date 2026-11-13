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

libP2PEvolution is truly peer to peer. It does not attempt to use 3rd party TURN servers to get around firewalls and NAT devices. It will attempt to use the following methods to communicate across the open Internet:
- Use automatic port forwarding
- Use IPv6
- Rely on the DHT service on the same port to keep packet routing consistent
- Use DHT to accurately determine the other peers' ports and attempt to open communication multiple times

## libP2PEvolution class

The core libP2PEvolution class must be instantiated in order to use this peer to peer functionality. Each instance of this class must be provided with a meeting ID. The meeting ID is central to the design of this library as it is used for everything:
- The meeting ID is a standard ASCII string that is provided by the calling code
- The meeting ID acts as the AES encryption key for all peer-to-peer communication
- A hash of the meeting ID is used by DHT to locate other clients in the same meeting
- The local UDP port number is derived from the meeting ID

A local UDP port (enabled for both ipv4 and ipv6 as appropriate) is opened by this class. All communication ultimately goes through this port. Automatic port forwarding will attempt to make this port accessible to the outside world.

If DHT is enabled, it will utilize the same UDP port. A switchboard function will determine whether incoming packets are intended for DHT or belongs to the underlying meeting (these will be encrypted). The DHT service will act as a normal DHT service but it is also responsible for finding other peers that belong to the same meeting. It will do this through the following steps:
1. Hash the meeting ID and register the meeting on the DHT
2. Verify that the nodes are responsive and that this peer is indeed registered
3. Come up with multiple hashes based on the meeting ID and register those as well. The idea is to make sure that some minimum number of DHT nodes are aware of this peer. Some hashes may not be viable due to attacks or bad luck so the idea is to spread the meeting across multiple hashes.
4. Rotate through this list of nodes and check to see if any other peers suddenly also appear. The idea is to be able to reconcile any network changes in just a few seconds by relying on a short list of DHT nodes to act as the matchmaking service.

## Peer class

When another peer is found, connection should immediately be established. This is done by sending handshake packets. All packets are encrypted so if the other peer does not share the same meeting ID then communication is impossible.

Once a connection is established then the calling code is handed a peer object. This represents the connected peer. The peer class is responsible for the following:
- Managing multiple connection objects. The 0 connection is responsible for peer communications and not used by the calling code.
- Determine the MTU. This will be done by sending various sized packets with the no-split flag to see which ones make it through.
- Reconcile changes to the IP address. If an IP does change, hopefully it happens on only one end and connectivity can immediately be re-established. IF this is not possible then the new IP will be found by checking the reliable DHT nodes.
- Maintain attributes that can be set by the calling code. These will be text strings. Setting an attribute on the local peer will cause it to be visible to other peers.
- Track all of the known IP/port addresses for this client. Determine which one is best by trying them all and prioritize the one that is fastest (lowest ping). If connectivity is lost when switch to another one.

## Connection class

The calling code can ask a peer object to provide a connection object. The connection object represents a single connection. The following connections will be supported:
- non-guaranteed/non-ordered - This is exactly like a typical UDP connection, packets can be sent but are not guaranteed to arrive or to arrive in any particular order
- guaranteed/non-ordered - This is a UDP style connection but all packets are guaranteed to arrive at their destination
- guaranteed/ordered - This is a TCP style socket that guarantees data arrives correctly
- data transfer - Internally this uses a TCP style socket but is meant for moving binary data. It supports both in-memory objects and files.

In addition, connections support multiple rate limiting features:
- A loss-packet approach guarantees that the connection will move data as fast as possible but at the cost of other traffic
- A ping based approach is much slower but allows data to be transferred so that it does not interfere with other traffic
- These can be set at both the peer and connection levels.
- Additional prioritization will also be managed by the peer class.

Internally the connections are stored in an array that is managed by the peer object.
- Each connection is actually 2 connections, the even bit holds the data of the calling code while the odd bit is used as a corresponding control channel.
- Connection 0/1 is a guaranteed/non-ordered channel created by default that is used by the peer class to manage itself
- The calling code may thus access up to 126 connections to another peer

