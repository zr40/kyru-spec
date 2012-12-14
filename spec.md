The network spec consists of two parts: peer-to-peer network metadata and transfer of Kyru objects.

Kyru uses [Protocol Buffers](http://code.google.com/p/protobuf/) to encode messages, metadata and objects.

Peer-to-peer network metadata
=============================
This part is based on [Kademlia](http://www.cs.rice.edu/Conferences/IPTPS02/109.pdf). Kyru uses the _k_ and α values suggested by the paper: _k_ = 20, α = 3.

Metadata
--------
Kyru uses Kademlia to obtain two types of metadata. Locating reachable nodes is handled by Kademlia itself. To locate nodes containing an object, Kyru stores and retrieves data in the Kademlia DHT.

Nodes and objects have a 160 bit ID. The distance between two IDs is defined as the bitwise exclusive-or of those IDs (low)

For a given object, the data has the following form:

* Node list
	* Node ID
	* IP address
	* Port
	* Timestamp

Every hour, a node publishes metadata about every stored object. This is done by sending a STORE message to the _k_ nodes closest to the object ID. The node list contains a single item containing information about the local node, and the timestamp is set to the current timestamp.

When a node receives a STORE message for an object ID, it will merge the node list into its own. This is done as follows:

* For each received node list item:
	* If the node ID is already present in the local list:
		* Set its timestamp to the highest timestamp of the local item and the received item.
	* If the node ID is not present in the local list:
		* Send a KEEP_OBJECT message with the object ID to the node specified in the item. If the node replies affirmatively with the correct node ID, add the item to the local list.

Once a node list item's timestamp is more than 24 hours in the past, it is removed.

Every hour, Kademlia republishes the metadata. This is done by sending a store message to the _k_ nodes closest to the object ID. The message contains the entire node list. (The timestamp isn't changed.) If the list doesn't fit in a single UDP packet, a random selection is sent.

Every hour, for all objects stored on a node, the node will send a KEEP_OBJECT message with the object ID to all nodes in the list. If that node doesn't reply affirmatively with the correct node ID, remove it from the list.

Kademlia RPC methods
--------------------
This is a summary of the Kademlia RPC methods.

Kademlia has four RPC methods: PING, FIND_NODE, FIND_VALUE and STORE. These messages are transmitted using UDP.

All messages include the sender's node ID.

All requests include a randomly generated message ID, which is returned in the response message.

If no response is received within 2 seconds of sending the request, the request is retransmitted. If again no response is received after 2 seconds, the node is removed. (A timeout of 2 seconds is long enough for nodes located anywhere in the world to respond, and fast enough to identify nodes with connectivity problems. Retransmission makes sure that occasional minor packet loss does not disrupt a single node.)

### PING

The PING method requests a node to respond. If it responds, the node is known to be alive at this point.

PING is also used to obtain the node ID from newly discovered nodes.

### FIND_NODE

The FIND_NODE method requests a node to return the _k_ nodes closest to the given object ID that the node knows about.

### FIND_VALUE

The FIND_VALUE method acts like FIND_NODE, but if the node contains the value, it will be returned instead.

### STORE

The store method allows nodes to store a metadata value at another node.

Kademlia operations
-------------------

### NODE_LOOKUP

NODE_LOOKUP traverses the network to locate the _k_ nodes closest to the object ID.

During this operation, α simultaneous requests may be performed.

* Create a sorted list of nodes of size k, containing the _k_ nodes closest to the object ID. The list is sorted by distance to the object ID, closest first.
* While the list contains nodes that haven’t been queried yet:
	* Take the node closest to the object ID that isn’t currently being queried for this operation.
	* Send a FIND_NODE request to this node.
	* If the response contains nodes closer to the object ID than the last node in the sorted list, add those nodes to the sorted list.
	* If the node doesn't respond, remove it from the list.

Once all requests have finished, the list will only contain nodes closest to the object ID, known to be alive.

### VALUE_LOOKUP

VALUE_LOOKUP traverses the network in order to retrieve a value.

The algorithm is the same as for NODE_LOOKUP, with the following differences:

* Instead of sending a FIND_NODE request, a FIND_VALUE request is sent.
* If a node containing the value is found, return the node and stop.

Kademlia would at this point store the requested object in one of the queried nodes that did not contain the object. Kyru does not do this.

Transfer of Kyru objects
========================

Terminology
-----------
*Discard (object)*
All data related to the object is removed from the node. After discarding the object, the node behaves as if it has never contained the object.

*Isolation (node)*
A node is isolated if it doesn't know about any other nodes.

Kyru RPC methods
----------------

### KEEP_OBJECT

KEEP_OBJECT indicates to a node that the object shouldn't be deleted. It's also used to query a node whether it contains an object. This message is transmitted using UDP.

Nodes receiving this message will set the access timestamp of the object to the current timestamp.

The response indicates whether the node contains the object.

### GET_OBJECT

GET_OBJECT requests a node to return an object. This message is transmitted using TCP.

The response contains the object.

### STORE_OBJECT

STORE_OBJECT requests a node to store an object. This message is transmitted using TCP.

The request contains the object's ID, type and SHA1 hash.

The response indicates whether the other node has accepted the object. (It might reject the object because it already contains it, it is full, or for other reasons.)

If the object is accepted, the node can now send the object.

Object definitions
==================

Common properties
-----------------

All objects share a set of common data fields and behavior.

These fields are part of every object's data:

* Object type
* Payload

Objects have the following behavior:

* If the access timestamp is more than 24 hours in the past, and the node is not isolated, the object is discarded.
* If the node leaves isolation, the access timestamp is set to the current timestamp.
* Replication algorithm:
	* The node executes NODE_LOOKUP for a random ID. This results in at most _k_ nodes.
	* For each node returned, one by one:
		* Execute STORE_OBJECT with the object on that node.
		* If the object was accepted, stop.

User object
-----------
Users have a private and public RSA key. The private key is never stored; it can only be derived from the username and password.

The keys are derived using the following algorithm:

* Using 10000 iterations of PBKDF2 with HMAC_SHA512, derive 2048 bits of data from the username (S parameter) and password (P parameter).
* Split this data into two parts, 1024 bits each.
* Interpret these parts as big integers.
* For both integers, find the next prime.
* Use these two integers as P and Q to calculate the RSA public and private keys.

This process takes around 1 second on a 2012 high-end desktop system.

The user object's ID is the SHA1 hash of the public key.

User objects have these fields as payload:

* File list
	* RSA signature for this file (over the SHA1 hash of the other fields)
	* File ID
	* RSA encrypted AES decryption key
	* AES encrypted file name (and IV)
	* Chunk list
		* Chunk object ID
* Deletion list
	* RSA signature for this deleted file (over the SHA1 hash of the other fields)
	* File ID

A file ID is a random 64 bit number, unique for the user. It's used only for deletions.

Chunks contain exactly 1 MiB, or less than 1 MiB for the last chunk. A file is first encrypted before being split into chunks.

User objects have the following behavior:

* Every hour, the node will execute the following algorithm for the user object itself, as well as all the chunk objects it contains:
	* Execute VALUE_LOOKUP for the object ID.
	* For all returned nodes, send KEEP_OBJECT for the object ID to that node. (This ensures that deleted data will expire, and data currently stored by a user will not.)
* When a node wants to send a different version of any already-known user object, it is merged in the following way:
	* Both nodes send the list of file IDs and deletion IDs to the other node.
	* If the local deletion list contains deletion file IDs which aren’t present in the received deletion list, send these deletion list items to the other node.
	* For all received deletion list items:
		* If the cryptographic signature is valid, add the item to the deletion list.
		* Add the deletion to the deletion list.
	* If the local file list contains file IDs which aren’t present in the received file list, send these file list items to the other node.
	* For all received file list items:
		* If the cryptographic signature is valid, add the item to the file list.
	* Delete all file list items whose file ID are on the deletion list.
	* Sort both lists by file ID. (The hash must be the same if the lists contain the same items. Sorting the lists ensures the order is the same on all nodes.)

Chunk object
------------

A chunk object's ID is the SHA1 hash of the entire object.

Chunk objects contain only the encrypted file piece as payload.

Chunk objects have the following behavior:

* If the SHA1 hash of the entire object does not match the object ID, the object is discarded.
