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
- List<String> currentTables = tableMonitorThread.tables();
+ List<String> whitelist     = config.getList(JdbcSourceConnectorConfig.TABLE_WHITELIST_CONFIG); 
+ Set<String>  whitelistSet  = whitelist.isEmpty() ? null : new HashSet<>(whitelist);
+ List<String> currentTables = new ArrayList<>(whitelistSet);
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
## Handle OS2200 driver lack of isValid impementation
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
## Map numeric types to numeric equivalents
**JdbcSourceConnectorConfig.java**
(*line 181*)
```diff
- public static final boolean NUMERIC_PRECISION_MAPPING_DEFAULT = false;
+ public static final boolean NUMERIC_PRECISION_MAPPING_DEFAULT = true;
```
## Map floating types based on their numeric scale
**DataConverter.java**
(*first instance of type NUMERIC*)
```diff
case Types.NUMERIC:
        if (mapNumerics) {
          int precision = metadata.getPrecision(col);
          if (metadata.getScale(col) == 0 && precision < 19) { // integer
            Schema schema;
            if (precision > 9) {
              schema = (optional) ? Schema.OPTIONAL_INT64_SCHEMA :
                      Schema.INT64_SCHEMA;
            } else if (precision > 4) {
              schema = (optional) ? Schema.OPTIONAL_INT32_SCHEMA :
                      Schema.INT32_SCHEMA;
            } else if (precision > 2) {
              schema = (optional) ? Schema.OPTIONAL_INT16_SCHEMA :
                      Schema.INT16_SCHEMA;
            } else {
              schema = (optional) ? Schema.OPTIONAL_INT8_SCHEMA :
                      Schema.INT8_SCHEMA;
            }
            builder.field(fieldName, schema);
            break;
          }
+	else if (metadata.getScale(col) > 0) { // double
+	  if (optional) {
+		builder.field(fieldName, Schema.OPTIONAL_FLOAT64_SCHEMA);
+		} else {
+		builder.field(fieldName, Schema.FLOAT64_SCHEMA);
+		}
+		break;
+       }
```
(*second instance of type NUMERIC*)
```diff
   case Types.NUMERIC:
        if (mapNumerics) {
          ResultSetMetaData metadata = resultSet.getMetaData();
          int precision = metadata.getPrecision(col);
          if (metadata.getScale(col) == 0 && precision < 19) { // integer
            if (precision > 9) {
              colValue = resultSet.getLong(col);
            } else if (precision > 4) {
              colValue = resultSet.getInt(col);
            } else if (precision > 2) {
              colValue = resultSet.getShort(col);
            } else {
              colValue = resultSet.getByte(col);
            }
            break;
          }
+         else if (metadata.getScale(col) > 0) { // double
+	    colValue = resultSet.getDouble(col);
+	    break;
+	  }
        }
```

