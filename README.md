# ssb-archive feature proposal<!-- omit from toc -->

The **Secure Scuttlebutt (SSB) community** is constantly seeking ways to improve the
performance and efficiency of the SSB system, particularly when it comes to
loading and processing large old feeds. To address this issue, we propose a new
feature called "ssb-archive".

The main goal of the ssb-archive feature is to **improve the user experience** when
loading and following a new feed, especially for large feeds, by allowing them
to be efficiently transmitted, scanned and processed. It also aims to **improve
the on-disk footprint** of the feeds by using a deterministic format that can be
shared as SSB blobs.

The ssb-archive feature is **designed to be optional**, so as not to impact existing
clients. It uses a FlatBuffer schema that is inspired by IPFS, Apache Parquet,
and other ideas, and employs dictionary encoding for compression and column
oriented encoding for key required fields. It avoids storing data that can be
derived from the message (like hash of the messages for instance), **maintain
the capability to rebuild the original message in its original feed format**, and
requires messages to be encoded in blocks of consecutive messages from a single
feed.

To implement the ssb-archive feature, a new message type called "archive" would
be introduced. The content of this message would contain a meta block called a
Bundle, which includes references to the archived MessageBlocks and
DictionaryBlocks. Clients with the ssb-archive feature would periodically create
and emit these messages to archive their old messages.

Clients without the ssb-archive feature would simply ignore that message, and
clients with the feature would be able to verify the integrity of the archived
messages without reloading when they have yet replicated them. This would
ensure that the security provided by the chain of feeds is not compromised.

Overall, the ssb-archive feature has the potential to greatly improve the
performance and efficiency of the SSB system, especially for large and
old feeds.

# Table of Content <!-- omit from toc -->

- [1. Design](#1-design)
  - [1.1. FlatBuffer schema](#11-flatbuffer-schema)
  - [1.2. MessageBlock and MessageBlockRef](#12-messageblock-and-messageblockref)
  - [1.3. Dictionary encoding](#13-dictionary-encoding)
  - [1.4. Format-specific data](#14-format-specific-data)
  - [1.5. Emitting "archive" messages](#15-emitting-archive-messages)
  - [1.6. Verifying and downloading archived messages](#16-verifying-and-downloading-archived-messages)
- [2. Compatibility](#2-compatibility)
- [3. Implementation](#3-implementation)
- [4. Benefits](#4-benefits)
- [5. Potential extensions](#5-potential-extensions)
  - [5.1 ssb-bipf2 as a general network and ondisk storage](#51-ssb-bipf2-as-a-general-network-and-ondisk-storage)
  - [5.2 Fromm ssb-archive to ssb-redux](#52-fromm-ssb-archive-to-ssb-redux)
  - [5.2 Random ideas](#52-random-ideas)


# 1. Design

## 1.1. FlatBuffer schema

The ssb-archive feature uses a FlatBuffer schema to encode the data that is
stored on disk and trasmitted as Blobs. We call this format ssb-bipf2. The
schema includes a number of tables, unions, and enums that are used to represent
the various data structures and data types. These include tables for storing
dictionaries, hash values, message and feed IDs, and blocks of messages,
as well as unions for representing feed-format-specific data and enums for
encoding data types. A full description of the schema can be found in the Annex.

## 1.2. MessageBlock and MessageBlockRef

The MessageBlockRef table is used to store a reference to a MessageBlock in
the ssb-bipf2 format. It contains the following fields:

* **blob_id**: A SHA hash that of the BlobId containing the MessageBlock.
* **sketch**: A sketch of the MessageBlock including its number of messages,
  first and last timestamps and last message hash.
* **previous**: A MessageBlockRef that stores a reference to the previous
  MessageBlock in the feed (optional).

The MessageBlock table is used to store blocks of consecutive messages from a
single feed. It contains the following fields:

* **sketch**: the same sketch as in the MessageBlockRef.
* **timestamp_deltas**: An array of the delta of timestamp field from the first
  message timestamp stored in the sketch, encoded using StreamVByte for
  efficient storage and searchability.
* **types**: An array of the types of the messages in the block, encoded using
  run length encoding and a dictionary for efficient scanning.
* **data**: A FormatSpecificBlockData union that stores the data for the messages
  in the block, in a feed-format-specific format.

Together, the MessageBlock and MessageBlockRef tables are used to store blocks
of consecutive messages from a single feed in a compact and efficient manner,
allowing them to be easily transmitted and processed.

## 1.3. Dictionary encoding

The ssb-bipf2 format uses dictionary encoding to compress the data stored in
the MessageBlock table. Dictionary encoding is a form of data compression that
uses a dictionary to map unique values to a smaller set of integers, which are
then used to represent the values in the data. This can significantly reduce
the size of the data, especially when there are many repeated values.

The ssb-bipf2 format supports two types of dictionaries: StringDictionary and
IntArrayDictionary. The StringDictionary is used to map string values to
integers, while the IntArrayDictionary is used to map arrays of integers to
integers.

Dictionaries are stored in SSB Blobs.

To use dictionary encoding, a dictionary must first be created and stored as a
separate block, using a DictionaryBlock table. The DictionaryBlock table
contains the following fields:

* **entries**: An array of the entries in the dictionary.
* **parent**: A DictionaryBlockRef that stores a reference to the previous
  dictionary (optional).

Using the parent field, dictionaries can be organized into a chain of
successive additions to the dictionary. It allows share common
dictionnaries between different feeds, and to update the dictionary when new
entries are added.

Once the dictionary has been created, it can be used to encode the data in the
MessageBlock table. The data is first mapped to the integers in the dictionary,
and then the integers are stored in place of the original data. This allows
the data to be compressed while still maintaining its original meaning.

## 1.4. Format-specific data

The ssb-bipf2 feature is designed to support multiple message formats, including
the classic format and any future formats that may be added. To handle these
different formats, the ssb-bipf2 feature uses a union called
FormatSpecificBlockData to store the data for the messages in a
feed-format-specific manner.

Currently, the only supported format is the classic format, which is used to
store messages in the original SSB format. To store data for classic format
messages, the ssb-bipf2 feature uses a table called ClassicMessageBlock. The
ClassicMessageBlock table contains the following fields:

* **sketch**: A sketch of the block including the public key of the author and the
  hash of the last message in the block.
* **object_signatures**: An array of the object signatures of the messages, encoded
  using run length encoding and a dictionary for efficient scanning.
  crypto_signatures: An array of the ed25519 signatures of the messages, encoded
  as a sequence of bytes.
* **messages**: An array of the remaining data for the messages, encoded using
  flexbuffers for compactness while maintaining ease of access.

In the future, additional feed formats may be added to the ssb-bipf2 format.
When
this happens, new entries will be added to the FormatSpecificBlockData union
to handle the data for these new formats.

In the classic format of SSB, messages are JSON objects and the hash and the
cryptographic signature of messages are derived from the JSON string
representation of the message. Therefore, it is important for the ssb-bipf2
binary format to maintain sufficient information about the structure of the JSON
object, in particular the order of the keys in objects, in order to be able to
rebuild the message in the original classic format.

In order to achieve compact storage while still being fast to scan and filter,
the ssb-bipf2 format uses a combination of flattening the JSON object,
dictionary encoding of keys, paths, and signatures, and storing data in a
column-oriented format.

Some message data are redundant and can be derived from other data in that
message block oriented format. The fields 'hash', 'sequence', 'previous',
'author', the actual SHA hash of the message are not stored in the message
block, but can be derived from the sketch fields.

After testing and investigation, it was found that there is a limited set of
unique keys and paths used in SSB messages, as well as a limited set of
sequences of keys and paths. This suggests that using flattening and
dictionary encoding can significantly reduce the size on disk and improve the
speed of scan and filter operations in the ssb-bipf2 format.

Certain paths, such as those in 'git-update', 'npm-packages', 'npm-publish',
'ssb-pkg-installer', and 'ssb-browser-app', contain JSON objects whose field
names are actually data and not structural information. In the ssb-bipf2
format, these fields are encoded as arrays of (key, value) tuples.

The ssb-bipf2 format stores the following fields for each message:

* A set of mandatory paths, including '**content.type**' and '**timestamp**', which
are stored in a column-oriented format for efficient retrieval.

* The **signature of the JSON object**, up to these excluded paths containing
arrays, is mapped to an integer using a dictionary of object signatures,
with each entry being an array of integers corresponding to the sequence of
paths themselves, which are encoded in a dictionary of paths, with each entry
being an array of integers corresponding to the sequence of field names,
which are encoded in a dictionary of keys. The integer representing the
object signature is stored in a column-oriented format at the Message Block
level.

* The data corresponding to the object signature is stored in an array
using FlexBuffer.

Overall, the ssb-bipf2 format allows for efficient and selective retrieval of
data while still maintaining the structure and meaning of the original message.

## 1.5. Emitting "archive" messages

To implement the ssb-archive proposal, a new message type called _"archive"_ would
be introduced. This message would be created and emitted by clients with the
ssb-archive feature to store blocks of old messages in a compact and efficient
format.

The process for creating and emitting an "archive" message would involve the
following steps:

* **Identify the messages to be archived**: The client would identify the old
  messages to be archived, typically by looking for messages that are no longer
  needed for usual daily processing but may be needed in the future.

* **Group the messages into blocks**: The client would then group the identified
  messages into blocks of consecutive messages from a single feed. Each block
  would be stored in a MessageBlock blobs as well as needed DictionaryBlocks
  blobs if not reused.

* **Create a Bundle**: The client would then create a Bundle, which is a meta block
  that includes references to the archived MessageBlocks and DictionaryBlocks.
  The Bundle would be stored in the content of the "archive" message.

* **Emit the "archive" message**: Finally, the client would emit the "archive"
  message containing the Bundle to the network. Other clients with the
  ssb-archive feature would receive and process the message, either rebuilding
  the blobs and validating that way the blobs created by the feed owner from
  their replica of the feed or retrieving the blobs and validating them from the
  content.

## 1.6. Verifying and downloading archived messages

Clients with the ssb-archive feature can verify the integrity of the archived
messages and download them using blobs. There are two cases to consider:

* **Clients that have already replicated all the messages archived by the feed
  owner**: These clients can rebuild the MessageBlock blobs and DictionaryBlocks
  blobs  
  using the same process as the feed owner. Once the blocks have been rebuilt,
  they can
  verify the integrity by comparing their SHA hash to the BlobId in Bundle
  emitted by the feed owner.

* **Clients that have not yet replicated all the messages in the Bundle**: These
  clients can retrieve the blobs and verify their integrity and consistency
  using
  the core verifications of the SSB protocol and the specific feed format.

Overall, the ssb-archive feature provides a way for clients to efficiently
verify and download archived messages, ensuring the security and integrity of
the data.

# 2. Compatibility

There are _no potential compatibility issues_ with the ssb-archive feature that
would affect existing clients. The ssb-archive feature introduces a new message
type called "archive" which existing clients can safely ignore if they do not
support it.

However, existing clients may want to directly use the ssb-bipf2 format on
disk to take advantage of the improved storage efficiency and query performance
offered by the ssb-bipf2 format without the need of extra indexes. To do this,
they will need to implement support for the ssb-bipf2 format in their storage
backend. This will allow them to store and retrieve messages in the ssb-bipf2
format, as well as use efficient sketching and querying features provided
by the format.

Overall, the ssb-archive feature is designed to be compatible with existing
clients, while also offering improved performance and efficiency for those that
choose to use it.

# 3. Implementation

The ssb-archive feature can be implemented in a few steps:

* Develop the **ssb-bipf2 format** for storing blocks of messages in a compact
  and efficient manner. This can be done in TypeScript and included in the
  current git repository.

    * The ssb-bipf2 reference implementation can implement the following
      operations:
        * **Building a MessageBlock** from a sequence of messages (and
          DictionaryBlocks if needed).
        * **Compacting/Merging one or more Bundles** of MessageBlocks and
          DictionaryBlocks (and optionally mapping the other Dictionaries).
        * **Querying a Bundle** of MessageBlocks on a set of criteria. To simplify
          the integration in Javascript-based stacks, the queries can be
          modelled
          after the ssb-db2 query format.

* Create the **"archive" message type** and implement a plugin for the current
  javascript SSB stack to handle the creation and emission of "archive"
  messages. This can also be done in TypeScript and included in the current git
  repository.

* Expose the following operations through the plugin:

    * **ssb-archive.configure**: This function allows the client application to
      configure the frequency at which the plugin checks for old messages to
      archive and the pre-conditions for archiving. The plugin will
      automatically check for old messages at the specified frequency, archive
      them using the ssb-bipf2 format, and emit an "archive" message. The plugin
      will also call registered callbacks to inform the rest of the client
      application of the archiving process. The pre-conditions for archiving are
      specified by the client application via a callback function that returns
      an integer. If the result is negative, the archiving is cancelled. If it
      is 0, the plugin starts the archiving immediately. If it is positive, the
      plugin waits the number of seconds and recalls the callback. This allows
      the application to delay the archiving process to not affect the user
      experience.
    * **ssb-archive.registerArchiveCallback**: This function allows the client
      application to flexibly add hooks to be informed when archives are built
      by the plugin. This can be used for integration with the existing storage
      backend, for example.

    * **ssb-archive.registerVerification**: This function receives 2 callback
      functions as parameters. When the ssb-archive plugin is configured, it
      monitors the arrival of SSB messages of type "archive" from other
      replicated feeds. Upon reception of such a message, it will call the first
      callback function, which is expected to return an integer. If the result
      is negative, the verification is cancelled. If the result is 0, the
      verification starts immediately. If the result is a positive n, the plugin
      will retry in n seconds.
      Once the verification is performed according to the verification process
      explained in the previous sections, the second callback is called with the
      result of the verification. This allows the application to take
      appropriate actions based on the result of the verification.

# 4. Benefits

The ssb-archive feature could offer a number of benefits, including:

* **Improved performance and efficiency**: By storing blocks of old messages in a
  compact and efficient format, the ssb-archive feature can significantly
  improve
  the performance and efficiency of SSB clients, particularly when accessing and
  processing large amounts of old data.

* **Reduced on-disk footprint**: The use of dictionary encoding and other
  compression techniques in the ssb-bipf2 format can significantly reduce the
  on-disk footprint of the archived messages, allowing clients to store more
  data
  using less disk space.

* **Efficient access to old messages**: The use of the ssb-bipf2 format and
  dictionary encoding allows clients to efficiently access and retrieve specific
  messages or blocks of messages from the archive, making it easier to find and
  use old data.

* **Reduced bandwidth usage**: The ssb-archive feature can also help reduce
  bandwidth usage when replicating new feeds, which can greatly benefit new
  users
  of SSB who may have limited bandwidth available.

# 5. Potential extensions

## 5.1 ssb-bipf2 as a general network and ondisk storage
1. The ssb-archive feature is driven by the feed owner's client application.
   This means that users replicating that feed and using a client application
   that supports the ssb-archive feature will only benefit from its features if
   the feed owner uses a client application that supports the ssb-archive
   feature and configures it to archive old messages. However, it is possible to
   **extend the usage of the ssb-bipf2 format to pubs servers** to facilitate the
   replication of old messages by new users. This can be achieved by allowing a
   pub server to store the messages in the ssb-bipf2 format and providing a way
   for clients to retrieve the messages in that format.

2. In a **direct replication scenario**, the ssb-bipf2 format can be used by peers
   to exchange messages in a compact and efficient manner if they share common
   dictionaries. The transcoding of messages from the ssb-bipf2 from one set of
   dictionaries to another can be implemented very efficiently. In fact,
   encoding messages in the ssb-bipf2 format should not be less performant and
   more demanding than JSON serialization while dramatically reducing bandwidth usage.

## 5.2 Fromm ssb-archive to ssb-redux

With the arrival of metafeed, it is likely that feeds will become more specialized and tailored to specific applications. This may involve defining schemas for message types and application-specific rules. For example, there may be feeds dedicated to messaging and social media, using a vocabulary and message types defined by ActivityPub. Additionally, there may be feeds for accounting that use ssb-tokens, or feeds for trust management and self-sovereign identity (ssb-DID).

A general way to view a feed is as a sequence of committed operations from the owner, which can be reduced to a view according to specific application rules, following the principles of the Event Sourcing design pattern as seen in P2PPanda. The ssb-archive commitment could be more focused on snapshot commitments, with application-specific rules for reduction and a unique data model for each block. For example, a feed for social media using ActivityPub vocabulary could drop previous versions of updated ActivityPub objects and deleted ActivityPub objects, while a feed for accounting could show the balances of exchanges.

Connected but not directly dependent, I believe that the evolution of the SSB propagation algorithm to better handle forked feeds is crucial for the future of the system. It is my personal opinion that SSB should develop a mechanism for detecting and propagating forks and give feed owners the option to reconcile branches by creating a join commitment message. I think this would greatly improve data management and replication across the network. It's a way to increase the robustness and reliability of the system.  My personal proposal to that are in https://gpicron.github.io/ssb-clock-spec/ (WIP)


## 5.2 Random ideas

1. **ssb-metafeed** comes with the notion of "stopping" a feed. This features 
   can also be combined with ssb-archive feature. 
