# Install new 19c database #

In this lab we will install the 19c database software and create a new 19c database (and listener) as target for the other upgrades.

## Disclaimer ##
The following is intended to outline our general product direction. It is intended for information purposes only, and may not be incorporated into any contract. It is not a commitment to deliver any material, code, or functionality, and should not be relied upon in making purchasing decisions. The development, release, and timing of any features or functionality described for Oracle’s products remains at the sole discretion of Oracle.

## Prerequisites ##

- You have access to the Upgrade to a 19c Hands-on-Lab client image


The below still needs to be formatted into MD format. This is planned for 24-MAR-20.

````
CREATE A NEW ORACLE HOME FOR 19C
Whether you are using your own environment or the provided Oracle Cloud environment during this workshop, we need to install a new Oracle 19c software tree on the environment. The following instructions are applicable to both personal environment (at work or on your laptop) and the Oracke Cloud environment provided during the workshop.
MAKE SURE YOU HAVE THE SOFTWARE AND UNZIP
The software downloaded from the Oracle network is a zipfile for your operating system/architecture. In the workshop environment, this zipfile has already been downloaded for you (see handout). If you brought your own environment, you already should have downloaded the software to a location of your choice.
--> LOCATE THE ORACLE 19C SOFTWARE ZIPFILE ON YOUR SYSTEM
In 19c, the location where you unzip the software wlll be used as your new Oracle Home. The running of the OUI will only register the software with the inventory (or will create one if none exist). So it is important to choose the location where you unzip the software carefully before running the OUI.
--> CREATE A NEW ORACLE HOME DIRECTORY FOR YOUR SOFTWARE
On the Oracle provided image, create the below mentioned directory. On your own environment, you can adopt to your company standard for creating a new Oracle Home.
[oracle@ws ~]$ mkdir -p /u01/app/oracle/product/19.0.0/dbhome_193

We can now use this new location to unzip our software
--> UNZIP THE ORACLE SOFTWARE ZIPFILE IN THE NEW DIRECTORY
[oracle@ws ~]$ cd /u01/app/oracle/product/19.0.0/dbhome_193

[oracle@ws dbhome_193]$ unzip /source/db_home_193_V982063.zip
...
  javavm/admin/classes.bin -> ../../javavm/jdk/jdk8/admin/classes.bin
  javavm/admin/libjtcjt.so -> ../../javavm/jdk/jdk8/admin/libjtcjt.so
  jdk/jre/bin/ControlPanel -> jcontrol
  javavm/admin/lfclasses.bin -> ../../javavm/jdk/jdk8/admin/lfclasses.bin
  javavm/lib/security/cacerts -> ../../../javavm/jdk/jdk8/lib/security/cacerts
  javavm/lib/sunjce_provider.jar -> ../../javavm/jdk/jdk8/lib/sunjce_provider.jar
  javavm/lib/security/README.txt -> ../../../javavm/jdk/jdk8/lib/security/README.txt
  javavm/lib/security/java.security -> ../../../javavm/jdk/jdk8/lib/security/java.security
  jdk/jre/lib/amd64/server/libjsig.so -> ../libjsig.so
INSTALL THE 19C PREINSTALL RPM ON THE SYSTEM
An easy way to make sure all system parameters are correct you can use the preinstall rpm package available for Linux. For non-Linux environments, please check the manual for the appropriate environment values.
We have already downloaded the preinstall rpm in the environment so you can simply install it.
 INSTALL THE 19C PREINSTALL RPM
[oracle@ws dbhome_193]$ sudo yum -y localinstall /source/oracle-database-preinstall-19c-1.0-1.el7.x86_64.rpm

Loaded plugins: langpacks, ulninfo
Examining /source/oracle-database-preinstall-19c-1.0-1.el7.x86_64.rpm: oracle-database-preinstall-19c-1.0-1.el7.x86_64
Marking /source/oracle-database-preinstall-19c-1.0-1.el7.x86_64.rpm to be installed
Resolving Dependencies
...
Running transaction
  Installing : oracle-database-preinstall-19c-1.0-1.el7.x86_64          1/1 
  Verifying  : oracle-database-preinstall-19c-1.0-1.el7.x86_64          1/1 

Installed:
  oracle-database-preinstall-19c.x86_64 0:1.0-1.el7                                                                    

Complete!

REGISTERING THE NEW ORACLE HOME TO THE INVENTORY AND CREATE NEW DATABASE
Before we can use the upgrade functionality, we need to run the runInstaller script to generate (relink and register) the new 19c Oracle Home. While running the OUI, we can choose to install the software only or to install the software and create a database. For various reasons, we will both install the software and create a new database in this lab.
--> NAVIGATE TO THE ORACLE HOME
[oracle@ws rdbms]$ cd /u01/app/oracle/product/19.0.0/dbhome_193
--> RUN THE RUNINSTALLER SCRIPT
[oracle@ws rdbms]$ ./runInstaller
The following screen should be visible on your (remote) desktop:
 
 KEEP 'CREATE AND CONFIGURE A SINGLE INSTANCE DATABASE' AND PRESS NEXT
 CHOOSE DESKTOP  CLASS AND PRESS NEXT
The desktop class will display 1 screen with all of the information required to create this type of database. If you think you need (for your local environment) other settings than displayed on the Desktop class screen, feel free to use the Server class.
For the Oracle provided Workshop environment, the Desktop class is enough. Make sure to check and change the following values in the various fields:

Oracle Base	: 	/u01/app/oracle (no changes)
Database File Location 	:	/u01/oradata (change this value)
Database Edition	: 	Enterprise Edition (no changes)
Characterset	: 	Unicode (no changes)
OSBDA group	: 	oinstall (no changes)
Global Database name	: 	CDB19 (change this value)
Password	: 	Welcome_123
Create as Container database	: 	Checked (no changes)
Pluggable database name	: 	PDB19C01 (change this value)
 
 ENTER THE CORRECT VALUES AND PRESS NEXT
Like previous installations, the root.sh script needs to be executed after the relinking and registration of the Oracle Home. The following screen lets you decide whether or not you want the OUI to do this for you. In this workshop environment, you can use the root password (OraclePTS#2019) for automatic execution of the root.sh script(s). 
For your local environment (at home), do what is applicable for your situation.
 
 CHECK OR UNCHECK ACCORDING TO YOUR ENVIRONMENT (WORKSHOP ENV=CHECKED) AND PRESS NEXT
The system will now start checking the prerequisites for the 19c installation.

 
 
You will now see a similar screen if any prerequisites for this installation have failed. This should not have happened in the provided workshop environment because of the 19c prerequisite rpm installation.
 
If all prerequisites have been checked and no warnings or errors can be found, the summary screen will be displayed:
 
 PRESS INSTALL TO START THE INSTALLATION
 
After a while, provided there are no issues during the install, the root.sh script needs to be executed. If you have entered the password for the root user in the OUI, permission will be asked to execute the scripts:
 
If you did not provide a root password or sudo information, a different window will be displayed. 
 
 FOR AUTOMATIC EXECUTION (PERMISSION SCREEN IS DISPLAYED): PRESS OK TO CONTINUE
 FOR MANUAL EXECUTION: EXECUTE THE ROOT.SH SCRIPT AS ROOT USER AND PRESS OK TO CONTINUE
[oracle@ws ~]$ sudo <directory and script in your window>

Performing root user operation.

The following environment variables are set as:
    ORACLE_OWNER= oracle
    ORACLE_HOME=  /u01/app/oracle/product/19.0.0/dbhome_193

Enter the full pathname of the local bin directory: [/usr/local/bin]: <press enter>
The contents of "dbhome" have not changed. No need to overwrite.
The contents of "oraenv" have not changed. No need to overwrite.
The contents of "coraenv" have not changed. No need to overwrite.

Entries will be added to the /etc/oratab file as needed by
Database Configuration Assistant when a database is created
Finished running generic part of root script.
Now product-specific root actions will be performed.
Oracle Trace File Analyzer (TFA - Standalone Mode) is available at :
    /u01/app/oracle/product/19.0.0/dbhome_193/bin/tfactl

Note :
1. tfactl will use TFA Service if that service is running and user has been granted access
2. tfactl will configure TFA Standalone Mode only if user has no access to TFA Service or TFA is not installed
 DATABASE WILL NOW BE CREATED, INFORM INSTRUCTOR
The installer will now start to create the new CDB database with its PDB. This can take between 20 and 40 minutes.
Please inform your instructor that you are waiting so he can keep track of the progress of the installs.
After the database creation has finished, the following screen (or similar) will be displayed:
 
 PRESS CLOSE TO END THE ORACLE UNIVERSAL INSTALLER
Your 19c Oracle Home has been created and the initial database (CDB19) has been started.
CHANGE DEFAULT MEMORY PARAMETERS
The OUI takes a certain percentage of the available memory in our environment as default SGA size. In our workshop environment, this is an SGA of 18G. We need the memory for other tasks later on so we will lower the memory usage of the new instance:
[oracle@ws ~]$ . oraenv
ORACLE_SID = [oracle] ? CDB19
The Oracle base remains unchanged with value /u01/app/oracle\

[oracle@ws ~]$ sqlplus / as sysdba

SQL*Plus: Release 19.0.0.0.0 - Production on Fri Mar 22 16:32:55 2019
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.


Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0

SQL> alter system set sga_max_size=4G scope=spfile;

System altered.

SQL> alter system set sga_target=4G scope=spfile;

System altered.

SQL> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.

SQL> startup
ORACLE instance started.

Total System Global Area 4294964632 bytes
Fixed Size                  9143704 bytes
Variable Size             855638016 bytes
Database Buffers         3422552064 bytes
Redo Buffers                7630848 bytes
Database mounted.
Database opened.

SQL> alter pluggable database all open;

Pluggable database altered.
````

** `End of Lab` **

