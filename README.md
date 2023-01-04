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

Second, Each group 
