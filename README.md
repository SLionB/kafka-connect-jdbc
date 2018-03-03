# Kafka Connect JDBC Connector for Unisys OS2200 RDMS Database

kafka-connect-os2200-jdbc is a [Kafka Connector](http://kafka.apache.org/documentation.html#connect)
for loading data from Unisys OS2200 JDBC-compatible RDMS database.

Documentation for this connector can be found at Unisystems

# Development

The following changes have been impemented arround the forked kafka-connect-jdbc 4.0.0-post repository:

1. Connector class is: gr.unisystems.connect.jdbc.OS2200JdbcSourceConnector
2. Connector is a source connector only
3. Supports input table names with ":" by replacing it with "-" in the output topic name
4. Remove double quotes around table names in calculated SQL query string
5. Identify OS2200 RDMS database from metadata and configure SQL statement to get current time as "select current_timestamp from RDMS.RDMS_DUMMY"
6. Remove checking database and table schema at startup and periodically with "TableMonitorThread" because of using a read only SQL user account - Whitelist of tables is added statically instead
7. Change default setting to not check fields for null validation with "VALIDATE_NON_NULL_DEFAULT = false" setting
8. Handle OS2200 driver lack of impementation for "connection.isValid()"  method by replacing it with "connection.isClosed()" method
9. Get additional field from the second version of each table using LEFT join on TL_CBSLG1_LOG_ID and TL_CBSLG2_LOG_ID fields
10. Change default setting to "NUMERIC_PRECISION_MAPPING_DEFAULT = true", to auto assign NUMERIC types to the correct types based on their sizes
11. Add support for NUMERIC types with "size > and presicion > 0" to be identified as DOUBLE types - All other not auto-identified types are mapped to STRING type instead of array of bytes that maps to BIGINT.
12. Correct timezone for date types to be "Europe/Athens" instead of "UTC" as log data are stored in RDMS in Greek time.

# Build

To build a development version you'll need a recent version of Kafka. You can build
kafka-connect-os200-jdbc with Maven using the standard lifecycle phases.

Install Open JDK

	yum install java-1.8.0-openjdk
	yum install java-1.8.0-openjdk-devel

Install Maven

	curl -O  http://www-eu.apache.org/dist/maven/maven-3/3.5.2/binaries/apache-maven-3.5.2-bin.tar.gz
	tar xzvf apache-maven-3.5.2-bin.tar.gz
	rm -f    apache-maven-3.5.2-bin.tar.gz
	mv       apache-maven-3.5.2 /opt/mvn
	/opt/mvn/bin/mvn -v
  
Clone repository

	cd /work  
	git clone https://github.com/SLionB/kafka-connect-os2200-jdbc.git
  
  
Build connector

	cd /work/kafka-connect-os2200-jdbc
	/opt/mvn/bin/mvn package -DskipTests -Dcheckstyle.skip
   

Deploy connector

	scp /work/kafka-connect-os2200-jdbc/target/kafka-connect-os2200-jdbc-4.0.1-SNAPSHOT.jar kafka1:/usr/share/java/kafka-connect-jdbc/ 
	scp /work/kafka-connect-os2200-jdbc/target/kafka-connect-os2200-jdbc-4.0.1-SNAPSHOT.jar kafka1:/usr/share/java/kafka-connect-jdbc/ 
	scp /work/kafka-connect-os2200-jdbc/target/kafka-connect-os2200-jdbc-4.0.1-SNAPSHOT.jar kafka1:/usr/share/java/kafka-connect-jdbc/
	
   
# License

The project is licensed under the Apache 2 license.
