# History of changes

## Support table names containing the character ":"
**TimestampIncrementingTableQuerier.java**(*line 214*)
```diff 
- topic = topicPrefix + name;
+ topic = topicPrefix + name.replaceAll(":","_"); 
```
## Remove double quotes around table names in calculated SQL query string
**TimestampIncrementingTableQuerier.java**(*line 92*)
```diff 
- builder.append(JdbcUtils.quoteString(name, quoteString))
+ builder.append(name); 
```

## Get currrent time for OS2200 RDMS database
**JdbcUtils.java**(*line 243*)
```diff
    String dbProduct = conn.getMetaData().getDatabaseProductName();
    if ("Oracle".equals(dbProduct)) {
      query = "select CURRENT_TIMESTAMP from dual";
    } else if ("Apache Derby".equals(dbProduct) || "DB2 UDB for AS/400".equals(dbProduct)) {
      query = "values(CURRENT_TIMESTAMP)";
+   } else if (dbProduct.toUpperCase().contains("RDMS")) { query = "select current_timestamp from RDMS.RDMS_DUMMY;";
    } else {
      query = "select CURRENT_TIMESTAMP;";
    }
```
## Avoid reading table schema at startup
**JdbcSourceConnectorConfig.java**
(*line 482*)
```diff
+     Set<String> whitelistSet  = new HashSet<>( (List<String>) config.get(JdbcSourceConnectorConfig.TABLE_WHITELIST_CONFIG) );  
      try (Connection db = DriverManager.getConnection(dbUrl, dbUser, dbPasswordStr)) {
-        return new LinkedList<Object>(JdbcUtils.getTables(db, schemaPattern, tableTypes));
+        return new LinkedList<Object>(whitelistSet);
      } catch (SQLException e) {
        throw new ConfigException("Couldn't open connection to " + dbUrl, e);
      }
```
## Avoid reading table schema periodically
**JdbcSourceConnector.java**
(*line 41*)
```diff
- import io.confluent.connect.jdbc.source.TableMonitorThread;
```
(*line 58*)
```diff
- private TableMonitorThread tableMonitorThread;
```
(*line 123*)
```diff
- 	  tableMonitorThread = new TableMonitorThread(
-	      cachedConnectionProvider,
-	      context,
-	      schemaPattern,
-	      tablePollMs,
-	      whitelistSet,
-	      blacklistSet,
-	      tableTypesSet
-	  );
-	  tableMonitorThread.start();
```
(*line 150*)
```diff
- 	  List<String> currentTables = tableMonitorThread.tables();
+	  List<String> whitelist     = config.getList(JdbcSourceConnectorConfig.TABLE_WHITELIST_CONFIG); 
+     Set<String>  whitelistSet  = whitelist.isEmpty() ? null : new HashSet<>(whitelist);
+	  List<String> currentTables = new ArrayList<>(whitelistSet);
```
(*line 166*)
```diff
-  log.info("Stopping table monitoring thread");
-  tableMonitorThread.shutdown();
-  try {
-    tableMonitorThread.join(MAX_TIMEOUT);
-  } catch (InterruptedException e) {
-  
- }
```
## Avoid checking non null validity by default
**JdbcSourceConnectorConfig.java**
(*line 181*)
```diff
- public static final boolean VALIDATE_NON_NULL_DEFAULT = true;
+ public static final boolean VALIDATE_NON_NULL_DEFAULT = false;
```
## Avoid checking non null validity by default
**CachedConnectionProvider.java**
(*line 65*)
```diff
  public synchronized Connection getValidConnection() {
      try {
            if (connection == null) {
            newConnection();
-           } else if (!connection.isValid(VALIDITY_CHECK_TIMEOUT_S)) {
+           } else if ( connection.isClosed() ) {
            log.info("Reconnecting to database because it is closed");
            newConnection();
            }
	    }
		catch (SQLException sqle) {
        log.info("Reconnecting to database because of the error : ", sqle);
	    connection = null;
	    return null;
        }
  return connection;
  }
```




