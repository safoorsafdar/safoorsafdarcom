---
title: How to delete million MySQL records
date: 2022-07-30T22:04:38.510Z
tags:
  - devops
  - aws
  - mysql
  - bigdelete
---
# How to delete million MySQL records

In this post, you will learn ways to delete records from the MySQL database server. This post goes through the methods to identify the solution to delete million of records with lesser time, fewer resources, and a 100% success rate. 

In a typical scenario, you will delete the records with the MySQL query;
`DELETE FROM * table where condition = ok` 

A million records to delete from a low-performance database. It will take too much time and be hectic at the same time due to the error-prone procedure. This might also chock the database. 

The first approach you could adopt is to divide the total number of records into chunks. LIMIT your delete query and increment the limit with the last deleted index.

For example, count the total number of records that you want to delete. It will help to prepare chunks of records. Verify deleted numbers of records after the activity. 

```mysql
SELECT COUNT(*) FROM quote WHERE DATE(updated_at) < '2022-07-01';
-- Count: 60000000
```

and, for example, delete all records that are older than 2022-07-01. Apply a limit to delete records in chunks. 

```
DELETE FROM quote WHERE DATE(updated_at) < '2022-07-01' LIMIT 10000000;
```

Keep executing the query by updating the `LIMIT` with a total number of chunks. In the mentioned example, I want to make a chunk of '10000000' and the total chunks would be six.

You will be able to remove junk records from your table but it will take too much time on the active table. Do not forget to verify the number of records you wanted to delete by running the `count(*)` query. 

The second solution is to make a temporary table. Switch it in and out, copy the last 30 day's data into it and drop the old table.

> :bulb: Enable your production site maintenance mode. After inserting the last 30 days of records, you should be able to disable maintenance mode. 

Create a temp table of your original table

```mysql
CREATE TABLE quote_new LIKE quote;
```

Switch in new empty temp table

```mysql
RENAME TABLE quote TO quote_old, quote_new TO quote;
```

Retrieve the last 30 days data and insert it into an empty table

```mysql
INSERT INTO quote SELECT * FROM quote_old
WHERE updated_at >= DATE_SUB(CURDATE(), INTERVAL 30 DAY);
```

In non-peak hours, drop the old table

```mysql
DROP TABLE quote_old;
```

Here are the advantages to doing DELETEs like this
- The table "quote" is switched with an empty table in a matter of seconds.
- The table "quote" is immediately available for new INSERTs
- The remaining 30 days are added back into "quote" while new INSERTs can take place.
- Dropping the old version of "quote" does not interfere with new INSERTs

> You can learn more about deleting large data records from http://mysql.rjweb.org/doc.php/deletebig

Any solution that tries to delete so much data in one transaction is going to overwhelm the rollback segment and cause a lot of performance problems.

A good tool to help is a `pt-archiver`. It performs incremental operations on moderate-sized batches of rows, as reliable as possible. pt-archiver can copy, move, or delete rows depending on options.

The documentation includes an example of deleting orphaned rows:
```shell
pt-archiver --source h=host,D=db,t=VALUE --purge \
  --where 'NOT EXISTS(SELECT * FROM `quote` WHERE DATE(updated_at) > '2022-07-01')' \
  --limit 10000000 --commit-each
```

Executing this will take longer to delete the data. But it will not use too much resources. And without interrupting service on your existing database. I have used it to delete millions of records of outdated data.

`pt-archiver` is part of the Percona Toolkit for MySQL, a free (GPL) set of scripts that help common tasks with MySQL and compatible databases.

