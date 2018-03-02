# Kafka Connect JDBC Connector for Unisys OS2200 RDMS Database

kafka-connect-os2200-jdbc is a [Kafka Connector](http://kafka.apache.org/documentation.html#connect)
for loading data from Unisys OS2200 JDBC-compatible RDMS database.

Documentation for this connector can be found at Unisystems

# Development

To build a development version you'll need a recent version of Kafka. You can build
kafka-connect-os200-jdbc with Maven using the standard lifecycle phases.

Connector class is: gr.unisystems.connect.jdbc.OS200JdbcSourceConnector.

Connector is a source connector only. 

Supports input table names with ":" by replacing it with "-" in the outoup topic name.

Remove double quotes around table name in SQL strings

Identify from metadata OS2200 RDMS database and configure SQL statement to get current time as select current_timestamp from RDMS.RDMS_DUMMY

Remove checking database and table schema at startup and periodically with TableMonitorThread because using read only SQL user account - Whitelist of tables is added statically

Remove support for checking  non null validation with VALIDATE_NON_NULL_DEFAULT = false setting

Handle OS2200 driver lack of impementation for connection.isValid()  method by replacing it with connection.isClosed()

Enable NUMERIC_PRECISION_MAPPING_DEFAULT = true, to auto assign NUMERIC types to numeric values

Add support for numeric types with size > and presicion > 0 to be identified as double types - All not auto identified types are mapped to strings instead of array of bytes.

Correct timezone for date types to be "Europe/Athens" instead of "UTC" as data are stored in RDMS in Greek time.


# License

The project is licensed under the Apache 2 license.
