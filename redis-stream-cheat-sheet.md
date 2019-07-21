# Redis Streams Cheat Sheet

## All commands

- `XINFO`: info on the status of streams, consumers or consumer groups
- `XADD`: add a message to a stream
- `DEL`: delete a stream
- `XGROUP`: manage consumer groups: create, destroy, set starting ID, as well as remove (dead) consumers from a group
- `XREAD`: read messages from a stream
- `XREADGROUP`: read from a stream as a consumer group
- `XRANGE`: read a range of messages from a stream
- `XREVRANGE`: read a range of messages from a stream in reverse order
- `XLEN`: get length of a stream (including all messages, already consumed or not)
- `XTRIM`: trim the length of a stream (i.e. delete older messages )
- `XDEL`: delete message(s) from a stream
- `XACK`: acknowledge a message has been consumed
- `XCLAIM`: claim pending messages (i.e. messages that have been consumed but not acknowledged)
- `XPENDING`: get info on pending messages (consumed but not acknowledged)

## Concepts

### Stream

A stream is like an append only log: messages are added to the end, and order is guaranteed, with no option to re-order.

### Consumer

A consumer reads from a stream. It will get messages in the order they were sent to the stream.
With the `XREAD` command, any consumer reading from a given stream will reads all messages in that stream.

```
          ┌──> A (consumer X)
F-E-D-C-B─┼──> A (consumer Y)
          └──> A (consumer Z)
```

### Consumer Groups

A consumer group is a way to distribute the message among multiple consumers. Each consumer in a given consumer group reading from a given stream (with the `XREADGROUP` command) will receive messages from that stream in a round robin fashion:

```
          ┌──> A (consumer X, group G)
H-G-F-E-D─┼──> B (consumer Y, group G)
          └──> C (consumer Z, group G)
```

### Message ID

Each message needs an ID which has to provide order.

Redis uses the timestamp in millisecond as default ID. It postfixes the timestamp with an index, which allows for multiple messages to be added within the same millisecond.

A default ID would look like: `1563467409672-0`

User defined IDs can also be used, but they need to follow the same pattern: `<numericID>-<index>`
When adding messages with user-defined IDs, the IDs need to be incrementing.

If a message was sent to a stream with ID `10-3`, the next message needs to have an ID with either an index greater than 3, or an numeridID greater than 10. Messages with ID `9-5` or `10-0` for example, will be rejected.

### How is it different from PUBLISH/SUBSCRIBE ?

`PUBLISH` and `SUBSCRIBE` commands allow for publishing to a set of consumers and consumers can subscribe to a publication. However, this pattern never stores the message: it merely transmists the message from a publisher to a set of connected consumers, but if a consumer is not connected, it will not receive the messages, and message sent while the consumer was disconnected cannot be replayed.

`PUBLISH` / `SUBSCRIBE` is a deliver `at-most-once` method.

In comparison, Redis Streams stores messages, so they can be fetched by consumers once they connect, and even replayed or transfered to other consumers in a consumer group, so it can be setup to deliver `at-least-once` (as well as `at-most-once`, if only reading new messages).


## Stream creation, deletion

### Stream Creation

There is no stream creation function per say.
Streams are created by adding messages to a stream.
If a stream does not exist, read commands will return nothing, or wait until a message arrives in case the BLOCK option was used, but it will not error if the key does not exist.

### Stream Deletion

Streams are just like other keys, so they can be deleted with the `DEL` command
```
DEL <stream_key>
```

## Adding messages 

Messages are added with the `XADD` command.
```
XADD key ID field string [field string ...]
```

The `*` character lets Redis generate IDs automatically.
```
XADD stream * myfield 10
```

Payload (field / value pairs) can be any number of fields, and does not need to be the same across all messages.

Payloads fields and values are string types.

#### Note
When reading payload, there is no option to read only specific fields. If large payloads are emitted but only part of them needs to be retrieved for processing, it may be more efficient to store the payload in a regular hash and only send the reference to the hash into the stream, so that consumers can read the reference from the stream and retrieve relevant payload field for processing.

## Reading messages

```
XREAD [COUNT count] [BLOCK milliseconds] STREAMS key [key ...] ID [ID ...]
```

### Reading from the beginning of the stream

Use `0` as ID to read from the beginning of the stream.

#### Note: 

Creating a message with ID `0-0` will result in an error. The smallest valid user-defined ID is `0-1` so that starting at `0-0` (exclusive) will indeed fetch all messages.

```
XREAD STREAMS mystream 0
```

#### Note:

```
XREAD STREAMS mystream 0-0
```
is equivalent to the above: `0` is a shorthand for `0-0`


Reading with `0` as ID will read all the messages in the stream, unless the `COUNT` option is used to limit the number of results.

#### Note:

The command allows for reading from multiple streams at once, but adding stream keynames and startin IDs

```
XREAD STREAMS mystream1 mystream2 mystream3 1563471187559-0 1563471187559-0 1563471187559-0
```

### Reading additional messages

To continue reading from a stream, the last read ID needs to be passed as the ID, so the `XREAD` command starts from that message ID (excluded)


To read additional messages from ID `1563470430931-0`, use:

```
XREAD STREAM mystream 1563470430931-0
```

This means the user needs to keep track of the last read message. It is good practice for the individual consumers to store that last read message in a separate key so that they can recall where they stopped last in cae of crash or restart.

### Reading a number of messages

To read a given number of messages at a time, use the `COUNT` option
```
XREAD COUNT 10 STREAM mystream 0
```

will fetch 10 messages at most (or less if there are less messages in the stream)

### Waiting for new messages

Sometimes, there are no messages in the stream, and constantly pulling for new messages in not efficient, while sleeping in between checks introduces latency in responding to a message. The `BLOCK` option registers the request on the Redis server and will cause the `XREAD` command to return with the next available message as soon as it arrives.

```
XREAD BLOCK 500 STREAM mystream $
```

The `$` ID option means read any new message since the command was started. This may potentially miss messages if messages were added before the command was ran.

This is typically used to fetch messages the first time (without reading from the beginning of the stream)

Further calls should use the last fetched message ID, to be sure not to miss any message.

```
XREAD BLOCK 500 STREAM mystream 1563470821993-0
```

#### Note:

```
XREAD COUNT 10 BLOCK 500 STREAM mystream 1563470821993-0
```

The `COUNT` option when used in combination with `BLOCK`, may not be intuitive:

`BLOCK` will cause the command to wait until the next message shows up, and return immediately, so if `COUNT` is set to `2` for example, the command still returns after a single message is added to the stream, with 1 message returned in total. If many messages are added to the stream in a transaction (with `MULTI`, multiple `XADD` and `EXEC`) then `XREAD` will read up to the number of messages specified by the `COUNT` option, since they were added simultaneously due to the transaction encapsulation of the `XADD` call.

## Manage Stream length
### Getting the number of messages in the stream

```
XLEN mystream
```

Gets the number of messages in the stream, in total, whether they were read or not by a consumer


### Trimming the length of a stream (removing oldest messages)

```
XTRIM mystream 1000
```
Will limit the length of the stream to 1000 messages, removing the oldest messages.

However, this may be an expensive operation, because the data is stored in a radix tree, and trimming to an exact length means scanning the whole tree down to the leaves.

If the exact number is not critical, approxiamte trimming is recommended:
```
XTRIM mystream ~1000
```
Will trim the stream to approximately 1000 message (leaving 1000 messages or more), by trimming a whole branch of the radix tree.

#### Note:
`XTRIM` doesn't care if messages were read or not. Automatic trimming should make sure messages were processed before trimming.

Another way to trim the stream is with the `MAXLEN` option when adding messages:

```
XADD mystream * field1 value1 MAXLEN 1000
```
Will trim the stream upon inserting the new message.


### Removing messages

`XDEL` is used to delete one or more messages by ID

While `XTRIM` and the `MAXLEN` option on `XADD` trim the oldest messages out of the stream, `XDEL` allows removing a message from the stream by ID

```
XDEL mystream 1563472498881-0
```

#### Note:
This allows some flexibility in skipping messages, or removing messages that were processed and no longer needed.

`XDEL` doesn't technically remove the message from memory like `XTRIM` does, but merely marks the message as deleted, so memory will only be reclaimed upon next garbage collection.

## Consumer Groups
### Creating a Consumer Group

Consumer Groups allows reading from a stream in a Round-Robin fashion, so as to spread the message processing workload across multiple workers.

To create a consumer group:

```
XGROUP CREATE key groupname id-or-$
```

```
XGROUP CREATE mystream group1 $
```
Will create a consumer group named `group1` for the stream `mystream`, starting consuming at new messages added after the group was created.

To start from the beginning of the stream used id `0` (or `0-0`)

### Setting start ID for a consumer group

Sometimes it is useful to be able to set the starting ID for a consumer group, such as for replaying a stream.

```
XGROUP SETID mystream groupname id-or$
```
Will set the ID at which the group should start reading.

### Remove a consumer group

```
XGROUP DESTROY mystream group1
```
Delete the consumer group `group1` for stream `mystream`, which also deletes all its consumers and pending entry lists (PEL)

### Adding a consumer to a group

Consumers are added to consumr groups simply by starting reading from the group:

```
XREADGROUP GROUP group consumer [COUNT count] [BLOCK milliseconds] [NOACK] STREAMS key [key ...] ID [ID ...]
```

```
XREADGROUP GROUP group1 myconsumer1 STREAMS mystream >
```
Adds a consumer `myconsumer1` to consumer group `group1` for stream `mystream` and reads messages never delivered to any consumer of the group.

it's a good idea to use a the `COUNT` option there to effectively spread the load across consumers, otherwise all messages will be delivered to the first consumer.

```
XREADGROUP GROUP group1 myconsumer1 COUNT 1 STREAMS mystream >
```

#### Note:

Using
```
XREADGROUP GROUP group1 myconsumer1 STREAMS mystream 0
```
Will return all messages that have been delivered to `consumer1` of consumer group `group1` but not acknowledged (i.e. messages in the pending entry list (PEL))

### Waiting for messages in a consumer group

The `BLOCK` option on `XREADGROUP` has the same effect as for the `XREAD` command.

Similarly `COUNT` act the same way as for the `XREAD` command.

### Acknowledge a processed message

With consumer groups, messages sent to consumers in a consumer group are also added to a Pending Entry List (PEL) which tracks which consumer is working on what message(s), and so that if the consumer crashes and never returns, these messages can be found and re-directed to another consumer.

For a consumer to acknowledge a message it uses the `XACK` command:

```
XACK mystream group1 1563472498881-0
```
acknowledges a specific message

Multiple IDs can be passed at once to acknowledge multiple messages at once.
This effectively removes the message from the PEL.

## Stream Information when using consuemr groups

### Getting information on the status of the stream

```
XINFO STREAM mystream
1) 1) "name"
   2) "group1"
   3) "consumers"
   4) (integer) 1
   5) "pending"
   6) (integer) 5
   7) "last-delivered-id"
   8) "1563471187559-2"
```

provides information on the size of the stream, number of consumer groups, first and last message IDs, and radix tree stats.

```
XINFO GROUPS mystream
1) 1) "name"
   2) "group1"
   3) "consumers"
   4) (integer) 1
   5) "pending"
   6) (integer) 5
   7) "last-delivered-id"
   8) "1563471187559-2"
2) 1) "name"
   2) "group2"
   3) "consumers"
   4) (integer) 0
   5) "pending"
   6) (integer) 0
   7) "last-delivered-id"
   8) "0-0"
```
Provides information on the consuemr groups: their name, number of consumers, number of pending messages and last delivered ID

```
XINFO CONSUMERS mystream group1
1) 1) "name"
   2) "consumer1"
   3) "pending"
   4) (integer) 5
   5) "idle"
   6) (integer) 281840

```
Provides information about consumers in a consumer group: name, number of pending messages and idle time

## Managing pending messages

### Get pending message information for a consumer

```
XPENDING key group [start end count] [consumer]
```

```
XPENDING mystream group1 - + 10
```
Returns a list of pending messages, with how long ago they were delivered, and the number of time they were delivered (which would be 1 if only consumed once, but would be higher than 1 if a message was re-delivered after a consumer restart or if the message was claimed by another consumer with `XCLAIM`)

### Claiming a message from another consumer

When a consuemr fails, the message that were consumed but not acknowledged stay in the Pending Entry List, and unless the consumer comes back online and re-reads the pending messages, they will not be processed.

Another consumer can claim those messages with `XCLAIM`

```
XCLAIM key group consumer min-idle-time ID [ID ...] [IDLE ms] [TIME ms-unix-time] [RETRYCOUNT count] [FORCE] [JUSTID]
```

