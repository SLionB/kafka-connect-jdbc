# History of changes

## Support table names containing the character ":"
**TimestampIncrementingTableQuerier.java**(*line 214*)

~~topic = topicPrefix + name;~~
> topic = topicPrefix + name.replaceAll(":","_"); 

## Remove double quotes around table names in calculated SQL query string
**TimestampIncrementingTableQuerier.java**(*line 92*)

~~builder.append(JdbcUtils.quoteString(name, quoteString));~~
> builder.append(name); 

