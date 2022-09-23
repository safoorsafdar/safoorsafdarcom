---
title: My journey to export/import large MySQL data
date: 2022-09-21T23:28:44.040Z
tags:
  - devops
  - aws
  - aws-rds
  - mysql
  - database-migration
---
In this post, you will learn about my journey to finding the best solution to restore large databases. And will go through the methods that I have POC. The aim is to migrate around 180GB of data on a new on-premises MySQL and AWS RDS.

**Little bit background of the activity...**

The database belongs to the Magento2 eCommerce store. And the need was to perform complete migration during the Magento2 upgrade.

The activity contains the following steps:

* Migrate Database, Migrate the database to a new MySQL server from the source database and to the AWS RDS.
* Deploy the latest application code
* Upgrade the Application

"Migrate Database" is the step, this post is going to review and find the best solution. The aim is to find the total duration of this activity. The post will attempt various POC to find the best possible solution.

The project is running on a Hybrid cloud model. The old application is on Rackspace with 70% traffic responsibility. And the new application is on AWS with AWS RDS database with 30% traffic responsibility for UAT.

As part of the Magento2 upgrade, the activity is to perform migration from source DB to AWS and Rackspace. After successful migration, AWS will take 70% of traffic. 

## **POC-1 Traditional way to export/import**

In the first attempt of the POC, let's use the `mysqldump` utility. Its a client utility that performs logical backups. And produce a set of SQL statements that can run on destination MySQL.

The performance of this solution was worst than expected.

The export of the 100GB plus DB data from the source DB took around 3.5 to 4 hours. 5 minutes to transfer the dump to the destination server. And the import to the MySQL server took around 4 hours without 100% success. It failed after 4 hours of time without any error.  And a few tables were missing. 

The spent time on this activity was around 8 hours which does not leave a place for other parts of the activity.

After the first POC, few actions were taken...

* Increase bandwidth to transfer data between the same network servers. and which reduced the transfer time to 3 minutes. Thanks to the Rackspace support team.
* As import was not successful. After some investigation, the destination server resources were at 100%. Most of the queries were timeouts. Added the alerts to track CPU & RAM usage at 80% & 90% on the destination server.
* Compare the configuration from source MySQL to destination. And configure the "insert buffer size", "packet size" and a couple more.

Any solution that tries to import so much data in one transaction will cause a lot of performance problems. So, decided to convert the whole dump into chunks.

## POC-1.1 Split dump into chunks

At this stage, I already have a dump of MySQL database via "mysqldump". I  can convert the complete database dump into chunks. For this purpose, the "bigsplit" shell script can help. You can learn more about this script at <http://blog.tty.nl/2011/12/28/splitting-a-database-dump>

```bash
#!/bin/bash

####
# Split MySQL dump SQL file into one file per table
# based on http://blog.tty.nl/2011/12/28/splitting-a-database-dump
####

if [ $# -lt 1 ] ; then
  echo "USAGE $0 DUMP_FILE [TABLE]"
  exit
fi

if [ $# -ge 2 ] ; then
  csplit -s -ftable $1 "/-- Table structure for table/" "%-- Table structure for table \`$2\`%" "/-- Table structure for table/" "%40103 SET TIME_ZONE=@OLD_TIME_ZONE%1"
else
  csplit -s -ftable $1 "/-- Table structure for table/" {*}
fi

[ $? -eq 0 ] || exit

mv table00 head

FILE=`ls -1 table* | tail -n 1`
if [ $# -ge 2 ] ; then
  mv $FILE foot
else
  csplit -b '%d' -s -f$FILE $FILE "/40103 SET TIME_ZONE=@OLD_TIME_ZONE/" {*}
  mv ${FILE}1 foot
fi

for FILE in `ls -1 table*`; do
  NAME=`head -n1 $FILE | cut -d$'\x60' -f2`
  cat head $FILE foot > "$NAME.sql"
done

rm head foot table*
```

The above-mentioned script utilizes 'csplit' Linux utility. Before execution of the script, make sure this package is available. The only downside of the script was to convert .gzip to .sql before executing the script.

After the execution of this script on dumped MySQL '.sql' file. It will have the number of '.sql' chunks based on the total number of the tables. In my case, it was 602 chunks.

I found a suggestion to import the MySQL dump with the source method. I prepared the shell script to run the chunks one by one.

```bash
#!/bin/sh
echo "ImportStarted, at: $(date +%F::%T)"
dbName="[Database name]"
userName="[MySQL User name]"
userPassword="[MySQL User password]"

batchSize="$(ls -1 *.sql | wc -l)"

echo "Batch Size:$batchSize"
for FILE in $(ls -1 sqldumps-chunks/*.sql); do
    #   NAME=`head -n1 $FILE | cut -d$'\x60' -f2`
    #   cat head $FILE foot > "$NAME.sql"
    ddl="set names utf8; "
    ddl="$ddl set global net_buffer_length=1000000;"
    ddl="$ddl set global max_allowed_packet=1000000000; "
    ddl="$ddl SET foreign_key_checks = 0; "
    ddl="$ddl SET UNIQUE_CHECKS = 0; "
    ddl="$ddl SET AUTOCOMMIT = 0; "
    # if your dump file does not create a database, select one
    ddl="$ddl USE $dbName; "
    ddl="$ddl source $FILE; "
    ddl="$ddl SET foreign_key_checks = 1; "
    ddl="$ddl SET UNIQUE_CHECKS = 1; "
    ddl="$ddl SET AUTOCOMMIT = 1; "
    ddl="$ddl COMMIT ; "

    fileSize="$(stat -c%s $FILE | numfmt --to=iec)"

    echo "ImportBatchStarted, for: $FILE, size: $fileSize"

    time mysql --user="$userName" --password="$userPassword" -e "$ddl"

    mysql --user="$userName" --password="$userPassword" -e "SELECT count(*) AS TOTALNUMBEROFTABLES FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA = '$dbName';" | grep -iv "TOTALNUMBEROFTABLES"
    echo "ImportBatchEnd, for: $FILE"
done
echo "ImportEnd, at: $(date +%F::%T)"
```

The duration of POC-1.1 was almost near to POC-1. The export got reduced to around 45 minutes or so. But upvote of 100% success rate on import. After the execution, the table count matched but the time duration was still beyond my expectations. I did not proceed with further verification of the data.

At this stage, the export time of MySQL dump is still the same. The transfer improved. And the improvement in import to 45 minutes or so. But the duration of the activity is still between 6 to 7 hours. That is not acceptable during the production activity.

So, still in peruse to find a more suitable solution.

## POC-2 Discovery of “MyDumper”

After researching different available open source tools, I landed on "MyDumper". This is a MySQL backup tool that offers 2 tools.

* “mydumper” which is responsible to export a consistent backup of MySQL databases
* “myloader” reads the backup from “mydumper”, connects it to the destination database, and imports the backup.

If you want to learn more about this tool, take a look at <https://github.com/mydumper/mydumper>.

**“MyDumper” installation for Centos 7**

```bash
release=$(curl -Ls -o /dev/null -w %{url_effective} <https://github.com/mydumper/mydumper/releases/latest> | cut -d'/' -f8)
yum install <https://github.com/mydumper/mydumper/releases/download/${release}/mydumper-${release:1}.el7.x86_64.rpm>
```

Above mentioned command is to set up a centos 7 base server for the “MyDumper”.

**Export data from the source**

"MyDumper" extracts the database data in parallel. And creates separate files from schemas and tables. That makes it easy to change them before restoring them.

```bash
# Remember to run the following commands in the screen as it is a long-running process.
time \
mydumper \
	--database="[Database name]" \
	--host="[Database host URL]" \
	--user="[Database user name]" \
	--password="[Database user password]" \
	--outputdir="./dumps-x2f1" \
	--rows=500000 \
	-G -E -R \
	--compress \
	--build-empty-files \
	--threads=30 \
	--compress-protocol \
	--verbose=3
```

It is important to track the execution time of command, for this I used `time`.

👆Assuming you want to export your DB data from the source into the `--outputdir`

* `--rows` will select a chunk of '500000' rows for a table.
* `--threads` will use multi-thread execution based on the available CPU. You can decide this number based on your available CPU and server load.
* `-G -E -R` will dump ‘Triggers’, ‘Events’, and ‘Routines’

With multi-thread and parallel execution of "mydumper" exporting the same database took around **8 to 10 minutes**. As compared to old POC, this is a huge improvement in time and operation.

Before importing this dump DB data,  clean “DEFINER” & “SQL SECURITY DEFINER”

```bash
cd ./dumps-x2f1
# Check if any schema files are using DEFINER, as files are compressed, we need to use zgrep to search
zgrep DEFINER *schema*
# Uncompress the schema files
find . -name "*schema*" | xargs gunzip
# Remove definers using sed
find . -name "*schema*" | xargs sed -i -e 's/DEFINER=`[A-Za-z0-9_]*`@`localhost`//g'
find . -name "*schema*" | xargs sed -i -e 's/SQL SECURITY DEFINER//g'
# Compress again
find . -name "*schema*" | xargs gzip
```

**Import the data into the destination DB**

"myloader" is the tool to import the dump into the database, below command is the example

```bash
time \
myloader \
	--database="[Database name]" \
	--password="[Database host URL]" \
	--user="[Database user name]" \
	--directory="./dumps-x2f1" \
	--queries-per-transaction=50000 \
	--threads=15 \
	--overwrite-tables \
	--compress-protocol \
	--verbose=3
```

👆Above mentioned command will start the import of the database with "myloader"

* `--queries-per-transaction` will execute queries per transaction.
* `--threads` will use multi-thread execution based on the available CPU. Choose your number of threads according to your available CPU.

With multi-thread execution of "myloader" to import dump, it took around **16 to 20 minutes.**  I gotta be honest, the whole process of migrating from one server to another, took around **30 minutes** which is a huge improvement over other solutions.

“MyDumper” utility is a clear winner to improve the performance of the operation which is acceptable during the production activity.

## Verification of data against source DB

The next step is to verify the database as compared to the source DB.  It is important to verify the data on both databases are the same after migration. Following are the few verifications you can perform:

* Check the tables count in the database
* Check the triggers count in the database
* Check the routines count in the database
* Check the events count in the database
* Check the rows count in the database

**Check the tables count in the database**

```sql
SELECT table_schema, COUNT(*) as tables_count FROM information_schema.tables group by table_schema;
```

**Check the triggers count in the database**

```sql
select trigger_schema, COUNT(*) as triggers_count from information_schema.triggers group by trigger_schema;
```

**Check the routines count in each database**

```sql
select routine_schema, COUNT(*) as routines_count from information_schema.routines group by routine_schema;
```

**Check the events count in each database**

```sql
select event_schema, COUNT(*) as events_count from information_schema.events group by event_schema;
```

**Check the rows count in database**

Check the rows count of all tables from the database. 

```sql
call COUNT_ROWS_COUNTS_BY_TABLE('prod24');
```

You need to create below mentioned procedure to be able to run above mentioned command.

```sql
DELIMITER $$

CREATE PROCEDURE `COUNT_ROWS_COUNTS_BY_TABLE`(dbName varchar(128))
BEGIN
DECLARE done INT DEFAULT 0;
DECLARE TNAME CHAR(255);

DECLARE table_names CURSOR for 
    SELECT CONCAT("`", TABLE_SCHEMA, "`.`", table_name, "`") FROM INFORMATION_SCHEMA.TABLES where TABLE_SCHEMA = dbName;

DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;

OPEN table_names;   

DROP TABLE IF EXISTS TABLES_ROWS_COUNTS;
CREATE TEMPORARY TABLE TABLES_ROWS_COUNTS 
  (
    TABLE_NAME CHAR(255),
    RECORD_COUNT INT
  ) ENGINE = MEMORY; 

WHILE done = 0 DO

  FETCH NEXT FROM table_names INTO TNAME;

   IF done = 0 THEN
    SET @SQL_TXT = CONCAT("INSERT INTO TABLES_ROWS_COUNTS(SELECT '" , TNAME  , "' AS TABLE_NAME, COUNT(*) AS RECORD_COUNT FROM ", TNAME, ")");

    PREPARE stmt_name FROM @SQL_TXT;
    EXECUTE stmt_name;
    DEALLOCATE PREPARE stmt_name;  
  END IF;

END WHILE;

CLOSE table_names;

SELECT * FROM TABLES_ROWS_COUNTS;

SELECT SUM(RECORD_COUNT) AS TOTAL_DATABASE_RECORD_CT FROM TABLES_ROWS_COUNTS;

END$$

DELIMITER ;
```

These steps will ensure that the destination database matches the source database. 

**Few pointers to migrating the database to AWS RDS**

The data is ready to import, let’s prepare the AWS RDS instance for faster restoration. As I mentioned after POC-1, there is a couple of configuration that need to apply to MySQL before import. Create an AWS Parameter group with  “insert buffer size", "packet size" and “innodb_” related configurations, and attach it to the RDS instance. 

You can also restore to a large RDS instance class to achieve faster restoration. And after the migration, you can lower down the RDS instance class.  

And you can perform a verification process like I have mentioned earlier to make sure the data is the same as the source database.

## Takeaways

* It is important to spend some time on research and POC to find a better solution that you can use during the actual activity.
* “MyDumper” is a multi-threaded and well-matured tool to use in production activity.
* Do not forget to tune MySQL-related configuration to get better performance.
* and last but not least, verification of the data after migration from the source to the destination should be part of the activity. It will help to ensure the integrity of the data.