# History of changes

## Modify existing connector
### 1. Remove parent dependencies from pom.xml and add manual dependenies

### 2. Support table names containing the character ":"
**TimestampIncrementingTableQuerier.java**(*line 214*)
```diff 
- topic = topicPrefix + name;
+ topic = topicPrefix + name.replaceAll(":","_"); 
```
### 3. Remove double quotes around table names in calculated SQL query string
**TimestampIncrementingTableQuerier.java**(*line 92*)
```diff 
- builder.append(JdbcUtils.quoteString(name, quoteString))
+ builder.append(name); 
```

### 4. Get currrent time for OS2200 RDMS database
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
### 5. Avoid reading table schema at startup
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
### 6. Avoid reading table schema periodically
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
### 7. Avoid checking non null validity by default
**JdbcSourceConnectorConfig.java**
(*line 181*)
```diff
- public static final boolean VALIDATE_NON_NULL_DEFAULT = true;
+ public static final boolean VALIDATE_NON_NULL_DEFAULT = false;
```
### 8. Handle OS2200 driver lack of isValid impementation
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
### 9. Implement LEFT JOIN on query string to get values from second set of tables
**TimestampIncrementingTableQuerier.java**
(*line 90*)
```diff
      case TABLE:
-               builder.append("SELECT * FROM ");
-		builder.append(name); // UniSystems change for OS2200
+		String name2 = name.replace("T_LOG_CBSLG1", "T_LOG_CBSLG2");
+		String incrementingColumn2 = incrementingColumn.replace("TL_CBSLG1", "TL_CBSLG2");
+		String appMessageColumn = "TL_CBSLG2_APP_MESSAGE";
+		builder.append("SELECT ");
+		builder.append(name + ".*,");
+		builder.append(name2 + "." + appMessageColumn);
+		builder.append(" FROM ");
+		builder.append(name + " LEFT JOIN " + name2);
+		builder.append(" ON " + name + "." + incrementingColumn + "=" + name2 + "." + incrementingColumn2);
        break;
```
(*timestampIncrementingWhereClause*)
```diff
    builder.append(" WHERE ");+
+   builder.append(name + ".");
    builder.append(JdbcUtils.quoteString(timestampColumn, quoteString));
    builder.append(" < ? AND ((");
+   builder.append(name + ".");
    builder.append(JdbcUtils.quoteString(timestampColumn, quoteString));
    builder.append(" = ? AND ");
+   builder.append(name + ".");
    builder.append(JdbcUtils.quoteString(incrementingColumn, quoteString));
    builder.append(" > ?");
    builder.append(") OR ");
+   builder.append(name + ".");
    builder.append(JdbcUtils.quoteString(timestampColumn, quoteString));
    builder.append(" > ?)");
    builder.append(" ORDER BY ");
+   builder.append(name + ".");// UniSystems change to handle joins
    builder.append(JdbcUtils.quoteString(timestampColumn, quoteString));
    builder.append(",");
+   builder.append(name + ".");// UniSystems change to handle joins
    builder.append(JdbcUtils.quoteString(incrementingColumn, quoteString));
    builder.append(" ASC");
```

### 10. Map numeric types to numeric equivalents
**JdbcSourceConnectorConfig.java**
(*line 181*)
```diff
- public static final boolean NUMERIC_PRECISION_MAPPING_DEFAULT = false;
+ public static final boolean NUMERIC_PRECISION_MAPPING_DEFAULT = true;
```
### 11. Map floating types based on their numeric scale
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
### 12. Map unmapped decimal types to string types
**DataConverter.java**
(*first instance of type DECIMAL*)
```diff
-	case Types.DECIMAL: {
-       int scale = metadata.getScale(col);
-       if (scale == -127) { //NUMBER without precision defined for OracleDB
-         scale = 127;
-       }
-       SchemaBuilder fieldBuilder = Decimal.builder(scale);
-       if (optional) {
-         fieldBuilder.optional();
-       }
-       builder.field(fieldName, fieldBuilder.build());
-       break;
-     }
+ case Types.DECIMAL: {
+ if (optional) {
+         builder.field(fieldName, Schema.OPTIONAL_STRING_SCHEMA);
+       } else {
+        builder.field(fieldName, Schema.STRING_SCHEMA);
+       }
+       break;
+ }
```
(*second instance of type DECIMAL*)
```diff
-	case Types.DECIMAL: {
-        ResultSetMetaData metadata = resultSet.getMetaData();
-        int scale = metadata.getScale(col);
-        if (scale == -127) {
-          scale = 127;
-        }
-        colValue = resultSet.getBigDecimal(col, scale);
-        break;
-      }
+	case Types.DECIMAL: {
+       colValue = resultSet.getString(col);
+       break;
+     }
```
### 13. Convert UTC time to Greek time
**DateTimeUtils.java**
(*line 25*)
```diff
public class DateTimeUtils {
-  public static final TimeZone UTC = TimeZone.getTimeZone("UTC");
+  public static final TimeZone UTC = TimeZone.getTimeZone("Europe/Athens");

  public static final ThreadLocal<Calendar> UTC_CALENDAR = new ThreadLocal<Calendar>() {
    @Override
    protected Calendar initialValue() {
-     return new GregorianCalendar(TimeZone.getTimeZone("UTC"));
+     return new GregorianCalendar(UTC);
    }
  };
```
## Create the new Connector
### Create new JAVA package gr.unisystems.connect as a copy of io.confluent.connect
### Rename JdbcSourceConnector class to OS2200JdbcSourceConnector on gr.unisystems.connect package
### Remove JdbcSinkConnector class from gr.unisystems.connect package

	

	
