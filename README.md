# Group-chat

## Description for Management state

In our case, it is obvious that the database is used primarily as a key-value store. No complex relational ops such as join are needed.
We could use NoSQL databases such as MongoDB.

First, group chat table should have following structure.

|Field Name|Type|Comment|
| ------------- |-------------|-----|
|groupId|String|Chat group Identifier|
|msgId|Integer|Message Id in group chat|
|timeStamp|DateTime|Message created time|
|contents|String|Message content|
|userId|String|Unique user's GUID|

Second, Group membership table structure

|Field Name|Type|Comment|
| ------------- |-------------|-----|
|groupId|String|Chat group Identifier|
|[userId]|String|Unique user's GUID|

## Description for Error Handling

If for some reason the WebSocket is cut off and the user is unreachable, all messages will be redirected to the notification service for a best-effort delivery (no guarantee of delivery, since the user might not have an internet connection)

We can handle error using error event handler(`onError`) functions for WebSocket. In `onError` function, we can determine the reason why error occurred and store those data in State and set notification flag as true.
On FE, if notification flag is set, get notification type and values from State and display them as Toast or attach error mark on center panel(message panel).

## Integrate WebSocket

### Communication Between Web Socket Servers

The first issue is how web socket servers communicate with each other. The easiest way is to use a separate, synchronous HTTP call for each message. However, two issues arise:

`1. No message ordering`: If two messages are sent in sequence, it is possible that the first one arrives super late. It will seem to the user that an unread message pops out above the latest message you just read, which is very confusing.

`2. Large number of request per second`: If each message is sent with a separate HTTP call, then the ingress traffic could overwhelm web socket Servers.
One possible solution is using queues as middleware. Each server has a dedicated incoming queue, which serves as a buffer and imposes ordering on messages.

One possible solution is using queues as middleware. Each server has a dedicated incoming queue, which serves as a buffer and imposes ordering on messages.

Although it looks like that queue is a better solution, I would argue that it’s a devil of its own:

* For large-scale applications such as Slack, there are tens of thousands of web socket servers. Maintains and scales the middleware is expensive and a challenge itself.
* If the consumer (web socket server) is down, we don’t want the messages to accumulate in the queue since the users will reconnect to a different server and initiate history catch-up. Servers come and go all the time, it is laborous to create/purge queues with them.
* If the consumer rejoins, what should we do with the stale messages in the queue? What messages to discard and what to process?

Here I propose a third approach that is based on synchronous HTTP calls:

To solve the ordering issue, we can annotate every message with a prevMsgID field. The recipient checks his local log and initiates history catch-ups when an inconsistency is found.

To reduce the number of messages exchanged between servers, we can implement some buffering algorithm on WebSocket servers — send accumulated messages, say, ~50 MS with randomized offsets (preventing everyone from sending messages at the same time).

### Handling Group Chat

Right now the current design broadcasts all group messages regardless of group size. This approach does not work well if the group is very large. Theoretically, there are two ways to handle group chats — pushing and pulling:

* pushing: The message is broadcast to other web socket servers who then push it to the client
* pulling: the client sends a request to HTTP servers for the latest messages.

The problem with pushing is that it converts one external request (one message) into many internal messages. This is called Write Amplification. If the group is large and active, pushing group messages will take up a tremendous amount of bandwidth.

The problem with pulling is that one message is read over and over again by different clients (Read Amplification). Going along with this approach will surely overwhelm the database.

My idea is that we can build a hybrid system with the above logic. For smaller groups or inactive groups, it is okay to do pushing since write amplification won’t stress out the servers. For very active and large groups, clients must query the HTTP server regularly for messages.

## Description for Components

1. Chat Service: each online user maintains a WebSocket connection with a WebSocket server in the Chat Service. Outgoing and incoming chat messages are exchanged here.
2. Web Service: It handles all RPC calls except send_message(). Users talk to this service for authentication, join/leave groups, etc. No WebSocket is needed here since all calls are client-initiated and HTTP-based.
3. Notification Service: When the user is offline, messages are pushed to external phone manufacturers’ notification servers.
4. Presence Service: When a user is typing or changes status, the Presence Service is responsible for figuring out who gets the push update.
5. User Mapping Service: Our chat service is globally distributed. We need to keep track of the server ID of the user’s session host.

![This is an image](https://github.com/emptyfist/Group-chat/blob/main/image.png)
