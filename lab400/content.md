# Upgrade to 19c Autonomous Database #

In this lab will be show the usage of a new tool called MV2ADB. This tool can, after completing the configuration file, execute all the steps to export, transport and import a database to the Oracle Autonomous Cloud.

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

## Check the current status of the source PDBs ##

Please log in as the Oracle user to check the status of the source database and pluggable databases.

### Set the environment for the 18c database ###

````
$ <copy>. oraenv</copy>
````
Enter the database 18c SID when requested:

````
ORACLE_SID = [oracle] ? DB18C
The Oracle base remains unchanged with value /u01/app/oracle
````

Now we can login as sysdba and check the status of the database

````
$ <copy>sqlplus / as sysdba</copy>
````

````
SQL*Plus: Release 18.0.0.0.0 - Production on Fri Mar 22 16:45:06 2019
Version 18.3.0.0.0

Copyright (c) 1982, 2018, Oracle.  All rights reserved.

Connected to:
Oracle Database 18c Enterprise Edition Release 18.0.0.0.0 - Production
Version 18.3.0.0.0
````

We can check the status of the PDBs in this container database using the command `show pdbs`:

````
SQL> <copy>show pdbs</copy>
````

Output similar to the following should be displayed:

````
    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 PDB18C01                       MOUNTED
````

So we have 1 PDB running in this environment in MOUNTED mode. In this lab we will migrate this PDB from the DB18C environment to the new 19c environment.
 
## Unplugging the source pluggable database ##

There are many ways to migrate a PDB to a new CDB. Some will keep the datafiles in place, other options will recreate the datafiles (double your storage size). If you are moving the datafiles to another storage system, you can for example use the Pluggable Archive option (all files will be put into 1 'ziptfile' with .pdb extension).
In the following lab, we will keep the datafiles on the same system but move them to another location. This can be done in a single command from SQL*Plus. After the migration to the new location, we can upgrade the PDB.

UNPLUG THE DATABASE INTO A PDB ARCHIVE
In the next example we assume you are already connected as sysdba to the DB18C database as described in the previous chapter.
 UNPLUG THE PDB INTO A PDB ARCHIVE
SQL> alter pluggable database PDB18C01 unplug into '/u01/PDB18C01.xml';

Pluggable database altered.

 EXIT THE ENVIRONMENT 

SQL> exit
Disconnected from Oracle Database 18c Enterprise Edition Release 18.0.0.0.0 - Production
Version 18.3.0.0.0

PLUG THE DATABASE INTO THE CDB19 ENVIRONMENT
 CHANGE THE ENVIRONMENT TO THE 19C ENVIRNMENT
[oracle@upgradews-4 ~]$ . oraenv
ORACLE_SID = [DB18C] ? CDB19
The Oracle base remains unchanged with value /u01/app/oracle
 LOG IN WITH SQL*PLUS AS SYSDBA
 [oracle@upgradews-4 ~]$ sqlplus / as sysdba

SQL*Plus: Release 19.0.0.0.0 - Production on Mon Mar 25 13:48:55 2019
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.


Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0
 PLUGIN THE 18C PDB AND MOVE THE DATAFILES
SQL> create pluggable database PDB18C01 using '/u01/PDB18C01.xml'
     move
     file_name_convert = ('/DB18C/','/CDB19/');

Pluggable database created.
The 'move' clause means that all relevant files should be moved to the new location. Using the 'file_name_convert', can you determine what the new location should be? Our files are now in a new location:
 QUERY THE V$DATAFILE TO CHECK THE FILE LOCATIONS
SQL> select name from v$datafile
     where name like '%18C01%';

NAME
--------------------------------------------------------------------------------
/u01/oradata/CDB19/PDB18C01/system01.dbf
/u01/oradata/CDB19/PDB18C01/sysaux01.dbf
/u01/oradata/CDB19/PDB18C01/undotbs01.dbf
/u01/oradata/CDB19/PDB18C01/users01.dbf
UPGRADE THE NEW PLUGGABLE DATABASE
 OPEN THE PDB IN UPGRADE MODE AND EXIT SQL*PLUS
SQL> alter pluggable database PDB18C01 open upgrade;

Pluggable database altered.

SQL> exit
We now need to upgrade the pluggable database as the PDB also contains a data dictionary and objects (which are still of the old version). When you upgrade a PDB, you use the commands you normally use with the Parallel Upgrade Utility. However, you also add the option -c PDBname to specify which PDB you are upgrading. Make sure to capitalize the name of your PDB while using this option.
 UPGRADE THE PDB USING THE PARALLEL UPGRADE UTILITY COMMAND CATCTL.PL
[oracle@ws ~]$ $ORACLE_HOME/perl/bin/perl $ORACLE_HOME/rdbms/admin/catctl.pl -d $ORACLE_HOME/rdbms/admin -c 'PDB18C01' -l $ORACLE_BASE catupgrd.sql

Argument list for [/u01/app/oracle/product/19.0.0/dbhome_193/rdbms/admin/catctl.pl]
For Oracle internal use only A = 0
Run in                       c = PDB18C01
Do not run in                C = 0
Input Directory              d = /u01/app/oracle/product/19.0.0/dbhome_193/rdbms/admin
...
After about 30 minutes the upgrade will be done:
...
Serial   Phase #:105  [PDB18C01] Files:1    Time: 1s
Serial   Phase #:106  [PDB18C01] Files:1    Time: 1s
Serial   Phase #:107  [PDB18C01] Files:1     Time: 0s

------------------------------------------------------
Phases [0-107]         End Time:[2019_03_25 15:36:25]
Container Lists Inclusion:[PDB18C01] Exclusion:[NONE]
------------------------------------------------------

Grand Total Time: 1839s [PDB18C01]

 LOG FILES: (/u01/app/oracle/catupgrdpdb18c01*.log)

Upgrade Summary Report Located in:
/u01/app/oracle/upg_summary.log

     Time: 1846s For PDB(s)

Grand Total Time: 1846s

 LOG FILES: (/u01/app/oracle/catupgrd*.log)


Grand Total Upgrade Time:    [0d:0h:30m:46s]
After the upgrade, the PDB will be left in a 'CLOSED' or 'MOUNT' state. Before we can work with the PDB we need to open it in normal read/write mode
 OPEN THE PDB IN READ/WRITE MODE
[oracle@upgradews-4 ~]$ sqlplus / as sysdba

SQL*Plus: Release 19.0.0.0.0 - Production on Mon Mar 25 15:57:09 2019
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.


Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0

SQL> alter pluggable database PDB18C01 open;

Pluggable database altered.
If no errors occur, we can login with SQL*Plus and check the invalid objects in the database
 LOGIN AS SYSDBA AND CHECK FOR INVALID OBJECTS
SQL> alter session set container=PDB18C01;

Session altered.

SQL> select COUNT(*) FROM obj$ WHERE status IN (4, 5, 6);

  COUNT(*)
----------
      1467
 IF THERE ARE INVALID OBJECTS, RECOMPILE THEM
SQL> @$ORACLE_HOME/rdbms/admin/utlrp.sql

Session altered.


TIMESTAMP
--------------------------------------------------------------------------------
COMP_TIMESTAMP UTLRP_BGN              2019-03-25 15:59:31

DOC>   The following PL/SQL block invokes UTL_RECOMP to recompile invalid
DOC>   objects in the database. Recompilation time is proportional to the
DOC>   number of invalid objects in the database, so this command may take

...

DOC> fixed before objects can compile successfully.
DOC> Note: Typical compilation errors (due to coding errors) are not
DOC>       logged into this table: they go into DBA_ERRORS instead.
DOC>#

ERRORS DURING RECOMPILATION
---------------------------
                          0


Function created.


PL/SQL procedure successfully completed.


Function dropped.


PL/SQL procedure successfully completed.
Your database is now migrated to a new $ORACLE_HOME and upgraded.


** `End of Lab` **