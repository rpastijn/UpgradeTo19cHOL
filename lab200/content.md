# Upgrade to 19c using the Autoupgrade tool #

In this lab we will leverage the Autoupgrade tool and upgrade an existing 12.1 CDB with 2 PDBs to 19c in a single command and configuration file.. 

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
 
## Prepare the Source database ##
One of the prerequisites for auto upgrade is that that database should be in **archive log mode**. The databases in our Workshop environment are not in archivelog mode, first step is to change this. We will use the preinstalled 12.1.0.2 database for this exercise (although we could have used the 11.2 database).

### Change source database into Archive Log mode ###

First, we set the environment variables for the 12.1 source environment:

````
$ <copy>. oraenv</copy>
````
````
ORACLE_SID = [oracle] ? <copy>DB121C</copy>
The Oracle base remains unchanged with value /u01/app/oracle
````

We can now log into SQL*Plus as sys user in sysdba mode:

````
$ <copy>sqlplus / as sysdba</copy>
````

After connecting, check the Archive Log status for this database:

````
SQL> <copy>archive log list</copy>
````
The result will most probably be the following:

````
Database log mode              No Archive Mode
Automatic archival             Disabled
Archive destination            USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     83
Current log sequence           85
````

If the output shows, like the above example, that the database is in `No Archive Mode`, enable Archive Log mode by executing the following steps:

1.	Shutdown the database
2.	Start database in mount mode
3.	Enable archive log mode
4.	Enable flashback mode
5.	Open (fully start) the database
6.	Check database status

````
SQL> <copy>shutdown immediate</copy>

Database closed.
Database dismounted.
ORACLE instance shut down.
````
````
SQL> <copy>startup mount</copy>
ORACLE instance started.

Total System Global Area 1603411968 bytes
Fixed Size                  2253664 bytes
Variable Size             469765280 bytes
Database Buffers         1124073472 bytes
Redo Buffers                7319552 bytes
Database mounted.
````
````
SQL> <copy>alter database archivelog;</copy>

Database altered.
````
````
SQL> <copy>alter database flashback on;</copy>

Database altered.
````
````
SQL> <copy>alter database open;</copy>

Database altered.
````
````
SQL> <copy>archive log list</copy>

Database log mode              Archive Mode
Automatic archival             Enabled
Archive destination            USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     83
Next log sequence to archive   85
Current log sequence           85
````

### Open all PDBS ###

The upgrade commands will only upgrade databases (and Pluggable Databases) that are in OPEN mode during the upgrade process. Therefore please check that all PDBs are in OPEN mode before you start the upgrade.

````
SQL> <copy>show pdbs</copy>
````

The following will probably be visible:
````
    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 DB121C01                       MOUNTED
         4 DB121C02                       MOUNTED
````

This means we need to open the PDBs first. Execute the following commands:

````
SQL> <copy>alter pluggable database all open;</copy>

Pluggable database altered.
````

And check again to see if it worked:

````
SQL> <copy>show pdbs</copy>

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 DB121C01                       READ WRITE NO
         4 DB121C02                       READ WRITE NO
````

## Create the Auto Upgrade Config file ##
 
The Auto Upgrade tool is part of the 19c Oracle Home distribution. Previous versions (<= 18.4) will need a separate download and setup from MyOracle Support under note 2485457.1. In this example we will only put one database into the configuration file but you can add as many databases as needed.

First, we will create a directory on the operating system where we can store Autoupgrade related files (and logfiles): 

````
$  <copy>mkdir -p /u01/autoupgrade</copy>
````
````
$ <copy>cd /u01/autoupgrade/</copy>
````

Now we can create a new configuration file for the Auto Upgrade tool. In this example the `vi` tool is used. If you are not familiar with `vi`, feel free to exchange the com
 CREATE SAMPLE CONFIG FILE FOR THE MIGRATION USING VI OR GEDIT
[oracle@ws autoupgrade]$ vi DB121C.cfg (or gedit DB121C.cfg)

Make sure the following lines are in the new file:
global.autoupg_log_dir=/u01/autoupgrade
upg1.dbname=DB121C
upg1.start_time=NOW
upg1.source_home=/u01/app/oracle/product/12.1.0/dbhome_121
upg1.target_home=/u01/app/oracle/product/19.0.0/dbhome_193
upg1.sid=DB121C
upg1.log_dir=/u01/autoupgrade
upg1.upgrade_node=localhost
upg1.target_version=19.3
upg1.run_utlrp=yes
upg1.timezone_upg=yes
 MAKE SURE YOU HAVE THE DB19C ENVIRONMENT IN YOUR VARIABLES
[oracle@ws autoupgrade]$ . oraenv
ORACLE_SID = [ORCL] ? CDB19
The Oracle base remains unchanged with value /u01/app/oracle
 EXECUTE THE ANALYZE PHASE OF THE AUTO UPGRADE TOOL WITH THE NEW CDB19 ENVIRONMENT
[oracle@ws autoupgrade]$ java -jar $ORACLE_HOME/rdbms/admin/autoupgrade.jar -config DB121C.cfg -mode analyze -noconsole

Autoupgrade tool launched with default options
+--------------------------------+
| Starting AutoUpgrade execution |
+--------------------------------+
1 databases will be analyzed

Job 100 for CDB121 FINISHED
In the parameter file we specified the location for the logfiles, in this case /u01/autoupgrade. After running of the command, the output stated a Job number (100). A new directory has been created in the logfile directory with this name; it contains all of the logfiles for this run. In the prechecks directory, the output of the prechecks are displayed in various formats:
[oracle@ws autoupgrade]$ cat 100/prechecks/db121c_preupgrade.log
[dbname]          [DB121C]
==========================================
[container]          [CDB$ROOT]
==========================================
[checkname]          CYCLE_NUMBER
[stage]              PRECHECKS
[fixup_available]    NO
[runfix]             N/A
[severity]           INFO
[rule]               The number of PDBs upgraded in parallel and the number of 
                     parallel processes per PDB can be adjusted as described in
                     Database Upgrade Guide.
[broken rule]        Using default parallel upgrade options, this CDB with 2 PDBs will
                     first upgrade the CDB$ROOT, and then upgrade at most 2 PDBs at
                     a time using 2 parallel processes per PDB.
[action]             No action needed.
----------------------------------------------------

[checkname]          DICTIONARY_STATS
[stage]              PRECHECKS
[fixup_available]    YES
[runfix]             YES
[severity]           RECOMMEND
[rule]               Dictionary statistics help the Oracle optimizer find efficient 
                     SQL execution plans and are essential for proper upgrade timing.
                     Oracle recommends gathering dictionary statistics in the last 24
                     hours before database upgrade.<br><br>For information on managing
                     optimizer statistics, refer to the 12.1.0.2 Oracle Database SQL
                     Tuning Guide.
[broken rule]        Dictionary statistics do not exist or are stale (not up-to-date).
[action]             Gather stale data dictionary statistics prior to database upgrade
                     in off-peak time using:
                     EXECUTE DBMS_STATS.GATHER_DICTIONARY_STATS;
In a regular upgrade situation, you can now check whether or not there are blocking issues with regards to your upgrade. The step executed here is basically the same as running the preupgrade.jar manually or through the DBUA.
 
EXECUTE THE UPGRADE
To continue with the full upgrade of the database(s) in the config file, run the same command but this time with the 'mode=deploy' option. 
 START THE FULL UPGRADE USING THE AUTOUPGRADE FEATURE
[oracle@ws autoupgrade]$ java -jar $ORACLE_HOME/rdbms/admin/autoupgrade.jar -config DB121C.cfg -mode deploy

Autoupgrade tool launched with default options
+--------------------------------+
| Starting AutoUpgrade execution |
+--------------------------------+
1 databases will be processed

upg>

The tool will now upgrade the requested databases in the background. You can request the status by executing the 'lsj' command:
 EXECUTE THE LSJ COMMAND
upg> lsj

+----+-------+---------+---------+-------+--------------+--------+--------+-----------------+
|JOB#|DB NAME|    STAGE|OPERATION| STATUS|    START TIME|END TIME| UPDATED|          MESSAGE|
+----+-------+---------+---------+-------+--------------+--------+--------+-----------------+
| 100| DB121C|PRECHECKS|PREPARING|RUNNING|19/04/19 11:51|     N/A|11:51:09|Remaining 198/246|
+----+-------+---------+---------+-------+--------------+--------+--------+-----------------+ 
Using the command prompt, you can do many things to control your upgrade. If there are any failure, you can correct the failures and restart the job or perhaps restore the environment to its original state.
On the operating system you can check the running of the upgrade as well. You can see for example, (in a second terminal window) you see that the regular tools are being used by the autoupgrade tool for the upgrade:
[oracle@ws dbs]$ ps -ef | grep perl

oracle   17211 11951  0 10:13 pts/4    00:00:01 /u01/app/oracle/product/19.0.0/dbhome_193/perl/bin/perl /u01/app/oracle/product/19.0.0/dbhome_193/rdbms/admin/catctl.pl -A -l /u01/autoupgrade/100/dbupgrade -i 20190321101158db112 -d /u01/app/oracle/product/19.0.0/dbhome_193/rdbms/admin catupgrd.sql
Please note that the perl command will only give you a result if the autoupgrade tool is actually running the perl scripts of course.
The logfiles in the /u01/autoupgrade/<job#> directory show you the progress as well for example:
[oracle@upgradews dbs]$ cd /u01/autoupgrade/<job#>

[oracle@upgradews 101]$ tail -f dbupgrade_<Press TAB>.log

2019-04-17 12:44:23.670 INFO Finished - Utilities.autoReadFileToAry 
2019-04-17 12:44:23.670 INFO Finished - Utilities.getPhaseNo 
2019-04-17 12:44:23.672 INFO [Upgrading] is [92%] completed for [db121c-pdb$seed] 
2019-04-17 12:47:23.747 INFO [Upgrading] is [92%] completed for [db121c-pdb$seed] 
+---------+---------------------------------------+
|CONTAINER|                             PERCENTAGE|
+---------+---------------------------------------+
| CDB$ROOT|SUCCESSFULLY UPGRADED [db121c-cdb$root]|
| PDB$SEED|                          UPGRADE [92%]|
| DB121C01|                          UPGRADE [92%]|
+---------+---------------------------------------+
The whole upgrade using the options chosen in this lab takes about 120 minutes
+--------------------------------+
| Starting AutoUpgrade execution |
+--------------------------------+
1 databases will be processed

Job 101 for DB121C FINISHED
After this your database should be upgraded to 19C. 
 CHECK IF THE DATABASE IS OPEN AND RUNNING:
[oracle@upgradews ~]$ . oraenv
ORACLE_SID = [CDB19] ? DB121C
The Oracle base remains unchanged with value /u01/app/oracle

[oracle@upgradews ~]$ sqlplus / as sysdba

SQL*Plus: Release 19.0.0.0.0 - Production on Fri Mar 22 16:38:53 2019
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.


Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0

SQL> select version from v$instance;

VERSION
-----------------
19.0.0.0.0
The autoupgrade tool was successful
You can check the logfiles for details regarding this upgrade now.
<end of lab>

