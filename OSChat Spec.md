# OSChat \- A mesh communication tool built over OSC. Miaow.

The purpose of this project is to build a resilient mesh communication tool for productions based on the OSC standard, with the ability to seamlessly bridge between isolated networks. Every node has the capability of being a gateway  and as close to zero configuration as possible for the end user is a design priority.

All communications are over the OSC protocol, sent to the local subnet broadcast IP on UDP port 22050 unless otherwise specified.

## The Protocol

  

### Design

`Node` \- unique on a given mesh, identified by a GUID. In practice, a node is an instance of software implementing this OSC protocol.

`Users` \- 0 or more may be on a given node. Identified by username. The same user may exist on multiple nodes although it is not recommended.

`Rooms` \- n or more may exist in a network where n is the number of users currently in the network. A room with the name equal to the name of a user is reserved for private messaging functionality.

Nodes are not expected to maintain any state throughout restarts.

### Minimum

`/oschat/message/<room> (str message, str user=‘Nobody’, int sequence=0)`

That’s it. As long as a client receives and sends messsges in this format, a full OSChat node will be able to process them and forward them on to further destinations. Clients who only implement this will show as permanently unconnectable on the client and will not appear in a room until they have broadcast a message to it.

Nodes connected to multiple interfaces MUST forward messages on additional interfaces only if they are the elected gateway to the other network, otherwise they MUST NOT forward messages.

Nodes must also discard duplicate messages received sequentially or within a specified timeout without forwarding. (duplicate \= where both user, message, and sequence number are identical.)

Nodes should save messages in memory as appropriate for scrollback if they have at least one active user in the room (depending on the specific use case)

### Recommended

`/oschat/ping/<user> (int sequence)`

Pings a user to determine if they are connectable. Nodes SHOULD implement a random backoff on retrying in order to prevent duplicate messages. Occurs roughly every 150s per user.

If a reply is not received within 1s, a node SHOULD mark the target user as unconnectable in the UI and start a timer to retry in  \~150s with a random offset applied to avoid excessive broadcasts. If a user is unconnectable for a configurable amount of time a node MAY permanently remove that user from its records and cease sending pings to that user entirely.

If a node receives a ping for a given user, even if no pong is received in response the ping timer MUST be restarted from zero.

Nodes connected to multiple interfaces MUST forward pings on additional interfaces only if they are the elected gateway to the other network, otherwise they MUST NOT forward pings.

Nodes must also discard duplicate pings received within a specified timeout without forwarding.

`/oschat/pong/<user> (int sequence)`

Responds to a ping for a specified user.

Nodes connected to multiple interfaces MUST forward pongs on additional interfaces only if they are the elected gateway to the other network, otherwise they MUST NOT forward pongs.

Nodes must also discard duplicate pongs received within a specified timeout without forwarding.

`/oschat/hello/<user> (int sequence)`

When connecting a user to a network a node SHOULD broadcast this message on all interfaces. Receiving nodes will add this user to their internal list of users and schedule pings as appropriate.

Nodes connected to multiple interfaces MUST forward hellos on additional interfaces only if they are the elected gateway to the other network, otherwise they MUST NOT forward hellos.

Nodes must also discard duplicate hellos received within a specified timeout without forwarding.

`/oschat/goodbye/<user> (int sequence)`

When disconnecting a user from a network a node SHOULD broadcast this message on all interfaces.

Receiving nodes will remove this user from their user list and cancel all pending pings for this user.

Nodes connected to multiple interfaces MUST forward goodbyes on additional interfaces only if they are the elected gateway to the other network, otherwise they MUST NOT forward goodbyes.

Nodes must also discard duplicate goodbyes received within a specified timeout without forwarding.

(May cause issues if a user is active on multiple nodes \- use reference tracking perhaps?)

`/oschat/list/users (int sequence, str payload=‘’)`

Sent without a payload this will prompt other nodes to reply with a list of all known connectable users. Receiving nodes are responsible for deduplicating strings.

If the list is empty the node MUST NOT reply.

Blah blah discard duplicates only forward if gateway.

`/oschat/list/rooms (int sequence, str payload=‘’)`

Sent without a payload this will prompt other nodes to reply with a list of all known rooms with at least one connectable user. Receiving nodes are responsible for deduplicating strings.

If the list is empty the node MUST NOT reply.

Blah blah discard duplicates only forward if gateway.

`/oschat/list/users/<room> (int sequence, str payload=‘’)`

Sent without a payload this will prompt other nodes in a given room to respond with a list of all known connectable users in the room. Receiving nodes are responsible for deduplicating strings.

If the list is empty the node MUST NOT reply.

Blah blah discard duplicates only forward if gateway.

`/oschat/join/<room>/<user> (int sequence)`

Prompts other nodes to add a user to their list of users in a given room.

Blah blah discard duplicates only forward if gateway.

`/oschat/leave/<room>/<user> (int sequence)`

Prompts other nodes to remove a user from their list of users in a given room.

Blah blah discard duplicates only forward if gateway.

### Mesh Gateways

Sorta BGP-esque, all nodes are capable of being gateways (peering) if they have multiple interfaces on different subnets. Nodes that only have a single interface do not interact with this except to elect gateways to other networks. Mesh commands are never forwarded.

Nodes store announces with a TTL of \~300s

`str node` \- a guid identifying the host sending the message.

`str network` \- the ipv4 or ipv6 network a host can reach in cidr notation

`int hops` \- the number of message gateways in the chain between two networks.

`/oschat/mesh/announce (str node, str network, int hops=null)`

1) sent on interface startup and every 150s subsequently by nodes with more than one interface to each interface \- hops must always \= 1 and network is a network not assigned to the interface this message is sent on.  

2) sent on other interfaces when an announce is received for a network not already routable \- node is replaced with this node's GUID and hops is incremented  
     
3) sent on other interfaces when an announce is received for a network already in the mesh and hops is not equal to or greater than a node-configurable max-hops value \- node is replaced with this node's GUID and hops is incremented by one  
     
4) when a node loses connectivity to a network it has previously announced, if it is not the active gateway, it should transmit this packet without the hops value \- other clients must remove it from their peers list \- if it is the active gateway it must resign instead

`/oschat/mesh/heartbeat (str node, str network)`

sent every 1s by the elected gateway for a given network as long as it is peered to that network

following an election of a gateway on a local subnet, the node that believes itself to be the new winner (or the tied nodes in the event of a tie) for that network should apply a random time offset before routing messages and beginning heartbeat transmission and if another heartbeat for that network is received within that timeframe, this node should cancel any routing and heartbeat transmission for the given network.

During this timeout, and only during this timeout, messages should be buffered for delivery following the start of normal routing.

`/oschat/mesh/elect (str node, str network)`

Periodically, on loss of a heartbeat signal (two or more missed) or when a resignation is received on a network from a given gateway, all nodes on the network should elect a gateway based on the data they have stored from announces. After ? milliseconds of no transmission of elections, the winning gateway, if it has changed, will begin routing messages to the peered network following the rules specified in the heartbeat section above.

Any node may force an election simply by sending this message \- other nodes on the local segment should then send their own election messages and reset any periodic timers/other statistics used to trigger one.

`/oschat/mesh/resign (str node, str network)`

if a gateway node is shutting down or loses peering to a remote network, it must resign from its role and force a new election. This will also immediately expire any announces for all networks from which it is resigning.  
