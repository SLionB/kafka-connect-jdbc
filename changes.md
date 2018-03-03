# History of changes

## Supporting table names containing the character ":"

** /src/main/java/io/confluent/connect/jdbc/source/TimestampIncrementingTableQuerier.java **
* line 214 *
```
~~topic = topicPrefix + name;~~
topic = topicPrefix + name.replaceAll(":","_"); 
```
