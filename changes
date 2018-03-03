
# Supporting table names containing the character ":"

* /src/main/java/io/confluent/connect/jdbc/source/TimestampIncrementingTableQuerier.java

```
from
		topic = topicPrefix + name;
to
		topic = topicPrefix + name.replaceAll(":","_"); 
```
