# Upgrade to 19c using Full Transportable Database #

In this lab we will use the Full (Cross Platform) Transportable Database functionality to migrate an existing 12.2 (single mode, non CDB architecture) to a 19c Pluggable database. 

## Disclaimer ##
The following is intended to outline our general product direction. It is intended for information purposes only, and may not be incorporated into any contract. It is not a commitment to deliver any material, code, or functionality, and should not be relied upon in making purchasing decisions. The development, release, and timing of any features or functionality described for Oracle’s products remains at the sole discretion of Oracle.

## Prerequisites ##

- You have access to the Upgrade to a 19c Hands-on-Lab client image
- A new 19c database has been created in this image
- All databases in the image are running

When in doubt or need to start the databases, please login as **oracle** user and execute the following command:

````
$ <copy>. oraenv</copy>
````
Please enter the SID of the 19c database that you have created in the first lab. In this example, the SID is **`19C`**
````
ORACLE_SID = [oracle] ? DB19C
The Oracle base has been set to /u01/app/oracle
````
Now execute the command to start all databases listed in the `/etc/oratab` file:

````
$ </copy>dbstart $ORACLE_HOME</copy>
````

The output should be similar to this:
````
Processing Database instance "DB112": log file /u01/app/oracle/product/11.2.0/dbhome_112/rdbms/log/startup.log
Processing Database instance "DB121C": log file /u01/app/oracle/product/12.1.0/dbhome_121/rdbms/log/startup.log
Processing Database instance "DB122": log file /u01/app/oracle/product/12.2.0/dbhome_122/rdbms/log/startup.log
Processing Database instance "DB18C": log file /u01/app/oracle/product/18.1.0/dbhome_18c/rdbms/log/startup.log
````

## Prepare the target 19c database ##

The FTTS functionality requires an existing database as a target. For this, we will log into the existing 19c instance and create a new Pluggable Database.

### Set the correct environment and login using SQL*Plus ###

````
$ <copy>. oraenv</copy>
````

Enter the SID for the 19c environment you already created in a previous lab:

````
ORACLE_SID = [DB112] ? CDB19
The Oracle base remains unchanged with value /u01/app/oracle
````

We can now login to the 19c environment. After login, we will create a new pluggable database as target:

````
$ <copy>sqlplus / as sysdba</copy>
````

### Create a new PDB called PDB19C02 ###

````
SQL> <copy>create pluggable database PDB19C02
     admin user admin identified by Welcome_123
     file_name_convert=('pdbseed','PDB19C02');</copy>
````

````
Pluggable database created.
````

In the above example we choose the location for the filenames by using the file_name_convert clause. Another option would have been setting the PDB_FILE_NAME_CONVERT init.ora parameter or have Oracle sort it out using Oracle Managed Files.

The files for this PDB have been created in /u01/oradata/CDB19C/PDB19C02

After creating the new PDB, we need to start it so it can be used as a target for our migration:

````
SQL> <copy>alter pluggable database PDB19C02 open;</copy>
````

````
Pluggable database altered.
````

## Prepare the target PDB ##

The migration described in this lab requires a directory object for Datapump and a databaselink to the source database. We will use our `/home/oracle` as temporary location for the Data Pump files.

````
SQL> <copy>alter session set container=PDB19C02;</copy>
````
````
SQL> <copy>create directory homedir as '/home/oracle';</copy>
````
````
SQL> <copy>grant read, write on directory homedir to system;</copy>
````
````
SQL> <copy>create public database link SOURCEDB
     connect to system identified by Welcome_123
     using '//localhost:1521/DB122';</copy>
````

We can check the database link to see if it works by querying a remote table:

````
SQL> <copy>select instance_name from v$instance@sourcedb;</copy>
````
````
INSTANCE_NAME
----------------
DB122
````

Just to be sure, make sure the user we need (and the contents of the source database) does not already exist in our target database. The user that exists in the source database (but should not exist in the target database) is the user PARKINGFINE and the table the schema contains is called PARKING_CITATIONS.

First check to see if the user exists in the target environment:
````
SQL> <copy>select table_name from dba_tables where owner='PARKINGFINE';</copy>
````
The correct answer should be:

    no rows selected

Second check, just to be sure, to see if the table exists in the source database:
````
SQL> <copy>select table_name from dba_tables@sourcedb where owner='PARKINGFINE';</copy>
````

The correct response should be:

````
TABLE_NAME
--------------------------------------------------------------------------------
PARKING_CITATIONS
````

As a quick check, we determine how much records are in the remote table:

````
SQL> <copy>select count(*) from PARKINGFINE.PARKING_CITATIONS@sourcedb;</copy>
````

The correct response should be:

````
  COUNT(*)
----------
   9060183
````

So the table and user exist in the source 12.2 database but not in the (target) PDB that we are connected to.

We can now exit SQL*Plus on the target system to continue with preparation of the source system.

````
SQL> <copy>exit</copy>
````

## Prepare the Source database ##

In order to run the full transportable operation we’ll have to take all data tablespaces into read-only mode – same procedure as we would do for a regular transportable tablespace operation. Once the tablespace (in this case it is just the USERS tablespace) is in read-only mode we can copy the file(s) to the target location. In our example, we only have 1 tablespace (USERS) that contains user data. If you execute a FTTS in another environment, make sure you identify all tablespaces !

Connect to the source 12.2 environment and start SQL*Plus as sysdba:

````
$ <copy>. oraenv</copy>
````
````
ORACLE_SID = [CDB19] ? <copy>DB122</copy>
The Oracle base remains unchanged with value /u01/app/oracle
````

We can now login the user sys as sysdba:

````
$ <copy>sqlplus / as sysdba</copy>
````

### Set tablespace to READ ONLY and determina datafiles ###

Change the tablespace USERS to READ ONLY to prepare it for transportation to the target database. Remember, in this example we only have data in the USERS tablespace. If you do this in another environment, determine all tablespaces applicable using dba_objects and dba_segments.

````
SQL> <copy>alter tablespace USERS READ ONLY;</copy>
````
    Tablespace altered.

We can now determine the datafiles that we need to copy to the target environment as part of the Transportable Tablespaces. We will only transport those tablespaces that contain user data:

````
SQL> <copy>select name 
     from v$datafile
     where ts# in (select ts# 
                   from v$tablespace 
                   where name='USERS');</copy>
````

The following should be the result:

````
NAME
--------------------------------------------------------------------------------
/u01/oradata/DB122/users01.dbf
````

Now that we have put the source tablespace to READ ONLY and know which datafiles we need to copy, we can copy the files to the target location. In this example, we will copy the files but we could also have moved the files. If you copy the files, you have a fall-back scenario in case something goes wrong (by simply changing the source tablespace to READ WRITE again). Downside is that you need extra diskspace to hold the copy of the datafiles.

Please exit SQL*Plus and disconnect from the source database.
````
SQL> <copy>exit</copy>
````
## Copy datafiles and import into 19c target PDB ##

First we simply copy the files to the location we will use for the 19c target PDB:

````
$ <copy>cp /u01/oradata/DB122/users01.dbf /u01/oradata/CDB19/PDB19C02</copy>
````

Now we can import the metadata of the database and the data (already copied and ready in the datafiles for the tablespace USERS in the new location) by executing a Datapump command. The Datapump import will be run through the database link you created earlier – thus no need for an file-based export or a dumpfile. Data Pump will take care of everything (currently except XDB and AWR) you need from the system tablepaces and move views, synonyms, trigger etc over to the target database (in our case: PDB19C02). Data Pump can do this beginning from Oracle 11.2.0.3 on the source side but will require an Oracle 12c database as target. This will work cross platform as well!!! (For cross-platform moves we would use RMAN backups and convert them if necessary.)

First we change our environment parameters back to 19c:

````
$ <copy>. oraenv</copy>
````
````
ORACLE_SID = [DB122] ? <copy>CDB19</copy>
The Oracle base remains unchanged with value /u01/app/oracle
````

We can now start the actual import process. 

````
$ <copy>impdp system/Welcome_123@//localhost:1521/PDB19C02 network_link=sourcedb \
        full=y transportable=always metrics=y exclude=statistics logfile=homedir:db122ToPdb.log \
        transport_datafiles='/u01/oradata/CDB19/PDB19C02/users01.dbf'</copy>
````

A similar output should be visible:

````
Import: Release 19.0.0.0.0 - Production on Wed Mar 20 13:18:36 2019
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

Connected to: Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Starting "SYSTEM"."SYS_IMPORT_FULL_01":  system/********@//localhost:1521/PDB19C02 network_link=sourcedb full=y t                                                                                                   ransportable=always metrics=y exclude=statistics logfile=homedir:db122ToPdb.log transport_datafiles=/u01/oradata/                                                                                                   CDB19/PDB19C02/users01.dbf
W-1 Startup took 1 seconds
W-1 Estimate in progress using BLOCKS method...
W-1 Processing object type DATABASE_EXPORT/PLUGTS_FULL/FULL/PLUGTS_TABLESPACE
W-1      Completed 0 PLUGTS_TABLESPACE objects in 5 seconds
...
W-1 Processing object type DATABASE_EXPORT/POST_SYSTEM_IMPCALLOUT/MARKER
W-1      Completed 1 MARKER objects in 2 seconds
W-1      Completed 1 DATABASE_EXPORT/EARLY_OPTIONS/VIEWS_AS_TABLES/TABLE_DATA objects in 0 seconds
W-1      Completed 51 DATABASE_EXPORT/NORMAL_OPTIONS/TABLE_DATA objects in 14 seconds
W-1      Completed 18 DATABASE_EXPORT/NORMAL_OPTIONS/VIEWS_AS_TABLES/TABLE_DATA objects in 20 seconds
W-1      Completed 7 DATABASE_EXPORT/SCHEMA/TABLE/TABLE_DATA objects in 3 seconds
Job "SYSTEM"."SYS_IMPORT_FULL_01" completed with 6 error(s) at Wed Mar 20 13:21:58 2019 elapsed 0 00:03:22
````

Usually you will find errors when using FTTS. By checking the logfile, you need to determine if the errors have a negative impact on your environment. In our migration, the errors should be only a few users that could not be created. The most important user we have in this database (PARKINGFINE) should have been migrated. We can check the target database to see if our table has been imported like it should:

````
$ <copy>sqlplus / as sysdba</copy>
````
````
SQL> <copy>alter session set container=PDB19C02;</copy>
````
````
SQL> <copy>select table_name from dba_tables where owner='PARKINGFINE';</copy>
````

The result should be this:
````
TABLE_NAME
--------------------------------------------------------------------------------
PARKING_CITATIONS
````

We can also check the number of records in the table by executing the following command:

````
SQL> <copy>select count(*) from PARKINGFINE.PARKING_CITATIONS;</copy>
````

The result should be this:

````
  COUNT(*)
----------
   9060183
````

The migration seems to be successful.

## Acknowledgements ##

- Author - Robert Pastijn, DB Dev Product Management, PTS EMEA - March 2020