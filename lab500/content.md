# Upgrade to 19c Autonomous Database #

Lab 500, version 11.0, Upgrade to 19c HOL, March 2020

## Contents ##

- Disclaimer
- Download and install required tools
	- Install the MV2ADB script
	- Check previously downloaded and installed Oracle Instant Client
	- Creating a new Autonomous ATP environment
- Create an Object Store Bucket
- Check Source schemas for compatibility
	- Package Source already downloaded
- Gathering required details
	- Gathering (source) DB Parameters
	- Gathering Expdp/Impdp Parameters
	- Gathering Object Store Properties
	- Gathering ADB Parameters
- Start the migration using the MV2ADB script
- Login and check the migrated database
 
## Disclaimer ##
The following is intended to outline our general product direction. It is intended for information purposes only, and may not be incorporated into any contract. It is not a commitment to deliver any material, code, or functionality, and should not be relied upon in making purchasing decisions. The development, release, and timing of any features or functionality described for Oracle’s products remains at the sole discretion of Oracle.

## Download and install the required tools ##

In this lab will be show the usage of a new tool called MV2ADB. This tool can, after completing the configuration file, execute all the steps to export, transport and import a database to the Oracle Autonomous Cloud.

### Install the MV2ADB script ###
The script that allows you easy migration to ADB can be downloaded from MyOracle Support through note **2463574.1**. In the Workshop environment we have already downloaded the tool for you in the `/source` directory.

> INSTALL THE MV2ADB TOOL AS ROOT USER

    [oracle@ws ~]$ sudo yum -y localinstall /source/mv2adb*.rpm
    Loaded plugins: langpacks, ulninfo
    Examining /source/mv2adb-2.0.1-40.noarch.rpm: mv2adb-2.0.1-40.noarch
    Marking /source/mv2adb-2.0.1-40.noarch.rpm to be installed
    Resolving Dependencies
    --> Running transaction check
    ---> Package mv2adb.noarch 0:2.0.1-40 will be installed
    ...
    Installing : mv2adb-2.0.1-40.noarch 1/1

    MV2ADB has been installed on /opt/mv2adb succesfully!

    Verifying : mv2adb-2.0.1-40.noarch 1/1

    Installed:
    mv2adb.noarch 0:2.0.1-40

    Complete!

Please note that the install script shows the location where the tool has been installed. In this case `/opt/mv2adb`. We need this later in the Lab.

> CHECK ALREADY INSTALLED ORACLE INSTANT CLIENT

We have already downloaded and unzipped the required files for the Oracle Instant Client. In the directory `/opt/instantclient` we have unzipped the base Instant Client, the SQL*Plus zip file and the Tools zip file. All have been downloaded from OTN.

Make sure the directory 19.6 exists and that the directory has contents

    [oracle@upgradews-4 ~]$ ls -l /opt/instantclient/18.3/
    total 235040
    -rwxr-xr-x 1 oracle oinstall 40617 Jun 28 2018 adrci
    -rw-r--r-- 1 oracle oinstall 1317 Jun 28 2018 BASIC_README
-rwxr-xr-x 1 oracle oinstall 1066102 Jun 28 2018 exp
-rwxr-xr-x 1 oracle oinstall 228672 Jun 28 2018 expdp
-rwxr-xr-x 1 oracle oinstall 57556 Jun 28 2018 genezi
...
CREATING A NEW AUTONOMOUS ATP ENVIRONMENT
In your Workshop hand-out you will see an Oracle Cloud Infrastructure URL like
https://console.<region>.oraclecloud.com
 NAVIGATE TO THE ORACLE CLOUD INFRASTRUCTURE CONSOLE AND LOGIN USING THE PROVIDED CREDENTIALS
 

 AFTER LOGIN, NAVIGATE TO THE AUTONOMOUS DATABASE SECTION
 
 MAKE SURE YOU HAVE SELECTED THE CORRECT COMPARTMENT
In the Lab handout you will see the compartment that you have access to. Please select this compartment using the dropdown box on the left side of the screen (ADB-COMPARTMENT-<city>-<date>)
 
 CREATE A NEW AUTONOMOUS DATABASE USING THE BLUE BUTTON
Workload Type	:	Autonomous Transaction Processing
Compartment	:	<keep value>
Display Name	:	ATP-<your-name>
Database Name	:	ATP<your-initials>
CPU Core Count	:	1
Storage (TB)	:	1
Autoscaling	: 	unchecked
Password	:	OraclePTS#2019
License Type	:	My Organization Already owns Oracle Database (etc..)
 ENTER REQUIRED DETAILS AND CREATE THE AUTONOMOUS DATABASE
This process should not take more than a few minutes to complete. 
You may continue with the Lab guide while the database is being created.
CREATE AN OBJECT STORE BUCKET
As we need to upload the export to the OCI environment, we need to create a location to do this. The MV2ADB script could create a new location but this would require the setup of the OCI Commandline tools. Since this Lab is using a more generic approach, we need to pre-create the directory (called Bucket).
 NAVIGATE TO OBJECT STORAGE IN THE OCI CONSOLE
 
 CREATE A NEW BUCKET WITH A UNIQUE NAME
 
Write down the name of the bucket as we will need it in our configuration file. The name of the bucket in the configuration file is case-sensitive.
CHECK SOURCE SCHEMAS FOR COMPATIBILITY
Not everything is supported in the Autonomous Database Cloud environment. To make sure you do not run into any issues, a tool called ADB Schema Advisor has been created. This PL/SQL Package can generate a report to show you any issues you might encounter before you actually execute the migration
PACKAGE SOURCE ALREADY DOWNLOADED
The ADB Schema advisor can be downloaded from MOS note 2462677.1 (Oracle Autonomous Database Schema Advisor). We have already downloaded the latest version to the /source directory in your client image.
Please note; in a regular environment, this package does not require SYS or SYSTEM user to be used. When installing it into a non-SYS and non-SYSTEM user, please check the manual for the exact installation steps. Very important are the GRANT statements required to give the package access to the information it needs.
 LOG INTO YOUR SOURCE 11.2 DATABASE AS SYS USER AND INSTALL THE ADB SCHEMA ADVISOR PACKAGE
[oracle@upgradews ~]$ . oraenv
ORACLE_SID = [oracle] ? DB112
The Oracle base remains unchanged with value /u01/app/oracle

[oracle@upgradews ~]$ sqlplus / as sysdba


SQL*Plus: Release 11.2.0.4.0 Production on Wed Apr 17 10:13:19 2019

Copyright (c) 1982, 2013, Oracle.  All rights reserved.


Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning option

SQL> @/source/adb_advisor.plb

Package created.


Package body created.

 EXECUTE THE ADB SCHEMA ADVISOR FOR SCHEMA HR
SQL> set serveroutput on
SQL> set linesize 1000
SQL> exec adb_advisor.report('HR','ATP')

The following is the first part of the possible output:
 
As you can see, there are some directory objects that cannot be migrated as ADB does not support access to the local filesystem (besides the DP_DUMP_DEST directory).
A second issue are 7 tables that apparently need changes before they can be imported. A little bit further down in the report, the issues are explained:
•	NOLOGGING options will be automatically changed to LOGGING options
•	Index Organized Tables (IOT) are not supported. You need a special option for IMPDP to change this during import.
 EXECUTE THE ADB SCHEMA ADVISOR FOR SCHEMA PARKINGFINE
 SQL> exec adb_advisor.report('PARKINGFINE','ATP')

======================================================================================
== ATP SCHEMA MIGRATION REPORT FOR PARKINGFINE
======================================================================================
--------------------------------------------------------------------------------------
-- ATP MIGRATION SUMMARY
--------------------------------------------------------------------------------------
                                                           Objects         Total
                           Object          Objects         Migrated        Objects
Object Type                Count           Not Migrated    With Changes    Migrated
-------------------------  --------------  --------------  --------------  ----------
DIRECTORY                  5               5               0               0
TABLE                      1               0               0               1
In this output, you can see that the schema PARKINGFINE has no issues with tables as it only contains a simple table with 9 million rows in it. We will use this schema to demonstrate the MV2ADB script.

GATHERING REQUIRED DETAILS
The configuration file for MV2ADB needs a series of parameters for export, upload of the dumpfile and import of the dumpfile. A full file with hints can be found in the /opt/mv2adb/conf directory. For this lab we will use only the parameters needed for a simple migration.
 CREATE A NEW FILE FOR THE CONFIGURATION
Please use your tool of choice (vi, desktop Text Editor etc) to create a new file. 
[oracle@ws ~]$ sudo vi /opt/mv2adb/conf/ATP.mv2adb.conf
or
[oracle@ws ~]$ sudo gedit /opt/mv2adb/conf/ATP.mv2adb.conf
Cut-and-paste the below parameters in this new document so that we can start entering the required data. At this moment, only copy-and-paste the below, we will make changes to the values in the following sections.
# DB Parameters
DB_CONSTRIG=//localhost:1521/DB112
SYSTEM_DB_PASSWORD=53152A9726C00647158CD4B1E103F1F2
SCHEMAS=PARKINGFINE
DUMPFILES=/tmp/DB112-<INITIALS>.dmp
OHOME=/u01/app/oracle/product/11.2.0/dbhome_112
ICHOME=/opt/instantclient/18.3

# Expdp/Impdp Parameters 
ENC_PASSWORD=53152A9726C00647158CD4B1E103F1F2
ENC_TYPE=AES256

# Object Store Properties 
BMC_HOST=
BMC_TENNANT=oraclepartnersas
BMC_BUCKET=
BMC_ID=
BMC_PASSWORD=

# ADB Parameters 
ADB_NAME=
ADB_PASSWORD=
CFILE=
GATHERING (SOURCE) DB PARAMETERS
The initial section is regarding the source database. Please enter the following information for the source environment. Since this is a Lab environment, we have pre-entered most of the DB parameters for you. Here is some information where you can find the details:
DB_CONSTRIG	Connecting string from the local system (where mv2adb is running) to the database instance that needs to be migrated
SYSTEM_DB_PASSWORD	Password for SYSTEM user for this source database
SCHEMAS	Schema's to be exported; only transport the schema's that you need, do not include any default schema's like HR, OE, SYSTEM, SYS etc as they already exist in ADB and might result in errors (or get ignored)
DUMPFILES	File system location of dumpfiles. If you want parallelism during export and import, specify as many files as you want the parallelism to be. Make sure the files are unique in the source but also in the destination (ATP) environment.
OHOME	The source database Oracle Home
IHOME	The installed Oracle Instant Client home (basic, SQL*Plus and Tools unzipped)
 MAKE SURE THE VALUE FOR THE DUMPFILES IS UNIQUE IN YOUR NEW CONFIG FILE
 CHANGE THE <INITIALS> IN THE DUMPFILES TO A UNIQUE VALUE
DUMPFILES=/tmp/DB112-<INITIALS>.dmp
GATHERING EXPDP/IMPDP PARAMETERS 
In this section you specify the encryption password and the encryption type for your export. To make sure your data cannot be compromised when exporting and uploading your data, the script requires a password.
ENC_PASSWORD	A password that will encrypt your datapump exports. Has nothing to do with any existing user or password. Please note that this password cannot be plain text. The password needs to be encrypted using the mv2adb binaries on your system
END_TYPE	Type of encryption of your data. The higher the number, the more encryption but also slower export/import. Options are  AES128, AES192 and AES256
 ENCRYPT THE ENCRYPTION PASSWORD AND PUT IT IN THE FILE
The password we will use for this Lab is Welcome_123.
[oracle@upgradews conf]$ /opt/mv2adb/mv2adb encpass

Please enter the password : Welcome_123
Please re-enter the password : Welcome_123

53152A9726C00647158CD4B1E103F1F2
Make sure YOUR encrypted password is entered in your new config file.
Example:
# Expdp/Impdp Parameters 
ENC_PASSWORD=53152A9729(..)647158CD4B1E103F1F2
ENC_TYPE=AES256
GATHERING OBJECT STORE PROPERTIES
The Autonomous database can only use dumpfiles uploaded to Swift compatible storage. The following parameters specify where the dumpfiles should be uploaded to after the export. This is also the location where the logfiles will be stored. Instead of the below SWIFT details, you can also choose to locally install the OCI Client and use that setup. See the example config for more information.
BMC_HOST	This is the Swift object storage URL for your environment. It usually has the form of https://swiftobjectstorage.<region>.oraclecloud.com where region is us-phoenix-1, eu-frankfurt-1 or similar
BMC_TENNANT	Name of your Tenancy. Be aware, for SWIFT we only use lower case
BMC_BUCKET	The name of an object storage bucket that will be used for this migration. You should have pre-created a bucket with a unique name UPGRADEBUCKET-<INITIALS>
BMC_ID	Your username in OCI (like ADB-<city>-<date>)
BMC_PASSWORD	The Swift Authentication Token encrypted using the mv2adb password encoder. The SWIFT authentication code can be found in your Workshop environment under /home/oracle/labs/keys
 LOCATE THE SWIFT AUTHENTICATION PASSWORD AND ENCRYPT USING THE MV2ADB TOOL
[oracle@ws ~]$ cat /home/oracle/auth_token.txt
loU5h0}(xxx)v(1.LcoPL

[oracle@ws ~]$ /opt/mv2adb/mv2adb encpass

Please enter the password : <cut-and-paste auth_key>
Please re-enter the password : <cut-and-paste auth_key>

E54C941DA0DBA8EB467DCC7F0C04(...)ED747D3AF6B6184BC173B78DE426CEBE4
 FILL IN ALL OF THE DETAILS FOR THE OBJECT STORE SETTINGS
Example (fill in our OWN details):
# Object Store Properties 
BMC_HOST=https://swiftobjectstorage.eu-frankfurt-1.oraclecloud.com
BMC_TENNANT=oraclepartnersas
BMC_BUCKET=ATP-MyInitials
BMC_ID=ADB-<city>-<date>
BMC_PASSWORD= E54C941DA0DBA8EB467DCC7F0C04(...)ED747D3AF6B6184BC173B78DE426CEBE4
GATHERING ADB PARAMETERS
During the gathering of the other parameters, your ADB environment should have been created. As a last step we will now gather the information needed for the last section
ADB_NAME	Name of your ADB instance. 
ADB_PASSWORD	Database ADMIN password
CFILE	Zipfile containing database credentials
First parameter requires the name of your created ADB environment. Navigate to the ADB Console and find the name of your database. This should be the 'DATABASE NAME' and not the 'Name' column:
 
Second parameter is the password you have entered while creating the Autonomous environment. If you have used the suggested password, it would be OraclePTS#2019. If you have chosen another password, you need to remember it.
 ENCRYPT YOUR DATABASE PASSWORD USING THE MV2ADB ENCRYPT OPTION
[oracle@upgradews-4 conf]$ /opt/mv2adb/mv2adb encpass

Please enter the password : <Your-Admin-Password>
Please re-enter the password : <Your-Admin-Password>
DE3D105A8E6F6A4D5E86EXSW6BC1D3BA
For the 3rd parameter we need to download something from the OCI console, the so-called Wallet file.
 NAVIGATE TO THE DETAILS OF YOUR ADB ENVIRONMENT
The result should be similar to this:
 
 CLICK ON THE BUTTON 'DB CONNECTION'
The following screen will be displayed:
 
 CLICK ON THE BUTTON 'DOWNLOAD' TO DOWNLOAD THE WALLET ZIP
In the following screen a password is requested. This is the password that protects the keystore inside the zipfile. For this exercise we will not be using this keystore so enter any random password twice.
 
 ENTER RANDOM PASSWORD AND PRESS 'DOWNLOAD'
Your zipfile will be downloaded to the default location /home/oracle/Downloads. Please note the name of the wallet.zip and enter this in your parameters.
# ADB Parameters 
ADB_NAME=ATPINIT
ADB_PASSWORD= DE3D105A8E6F6A4D5E8XXXSW6BC1D3BA
CFILE=/home/oracle/Downloads/Wallet_ATPINIT.zip
 MAKE SURE ALL PARAMETERS ARE ENTERED AND SAVE THE FILE TO /HOME/ORACLE/ATP.MV2ADB.CFG
The following is an example file for the MV2ADB setup:
# DB Parameters
DB_CONSTRIG=//localhost:1521/DB112
SYSTEM_DB_PASSWORD=53152A9726C00647158CD4B1E103F1F2
SCHEMAS=PARKINGFINE
DUMPFILES=/tmp/DB112-RP.dmp
OHOME=/u01/app/oracle/product/11.2.0/dbhome_112
ICHOME=/opt/instantclient/18.3

# Expdp/Impdp Parameters
ENC_PASSWORD=53152A9726C00647158CD4B1E103F1F2
ENC_TYPE=AES256

# Object Store Properties
BMC_HOST=https://swiftobjectstorage.eu-frankfurt-1.oraclecloud.com
BMC_TENNANT=oraclepartnersas
BMC_BUCKET=UPGRADEBUCKET-<INITIALS>
BMC_ID=ADB-<CITY>-<DATE>
BMC_PASSWORD=E54C941DA0DXXXB467DCC7F0C04A07ED747D3AF6B6184BC173B78DE426CEBE4

# ADB Parameters
ADB_NAME=ATPINIT
ADB_PASSWORD=DE3D105A8E6F6AXXXE86E4C06BC1D3BA
CFILE=/home/oracle/Downloads/Wallet_<DBNAME>.zip
 MAKE SURE ALL PARAMETERS ARE CORRECT, SAVE THE FILE AND EXIT THE EDITOR
The file should be in the /opt/mv2adb/conf directory and is called ATP.mv2adb.conf
[oracle@ws ~]$ ls -l /opt/mv2adb/conf
total 12
-rw-r--r-- 1 root root  471 May 17 13:13 ATP.mv2adb.conf
-rwxr-xr-x 1 root root 5159 Dec 28 12:52 DBNAME.mv2adb.cfg
START THE MIGRATION USING THE MV2ADB SCRIPT
Now we can start the actual migration by starting the MV2ADB tool using the proper options.
 START THE MV2ADB SCRIPT USING THE CONFIGURATIONFILE YOU JUST CREATED
[oracle@ws ~]$ sudo /opt/mv2adb/mv2adb auto -conf /opt/mv2adb/conf/ATP.mv2adb.conf
INFO: 2019-03-27 14:08:27: Please check the logfile '/opt/mv2adb/out/log/mv2adb_12753.log' for more details


--------------------------------------------------------
mv2adb - Move data to Oracle Autonomous Database
Author: Ruggero Citton <ruggero.citton@oracle.com>
RAC Pack, Cloud Innovation and Solution Engineering Team
Copyright (c) 1982-2019 Oracle and/or its affiliates.
Version: 2.0.1-29
--------------------------------------------------------

INFO: 2019-03-27 14:08:27: Reading the configuration file '/opt/mv2adb/conf/ATP.mv2adb.conf'
INFO: 2019-03-27 14:08:28: Checking schemas on source DB
...
INFO: 2019-03-27 14:08:54: ...loading '/tmp/DB112-RP_01.dmp' into bucket 'UPGRADEBUCKET-RP'
SUCCESS: 2019-03-27 14:09:27: ...file '/tmp/DB112-RP_01.dmp' uploaded on 'UPGRADEBUCKET-RP' successfully
SUCCESS: 2019-03-27 14:09:27: Upload of '1' dumps over Oracle Object Store complete successfully

INFO: 2019-03-27 14:09:27: Performing impdp into ADB...
INFO: 2019-03-27 14:09:27: Step1 - ...drop Object Store Credential
INFO: 2019-03-27 14:09:29: Step2 - ...creating Object Store Credential
INFO: 2019-03-27 14:09:36: Step3 - ...executing import datapump to ADB
INFO: 2019-03-27 14:12:42: Moving impdp log 'mv2adb_impdp_20190327-140936.log' to Object Store
SUCCESS: 2019-03-27 14:12:43: Impdp to ADB 'ATPINIT' executed successfully
After about 10 minutes, all the steps should have been executed successfully. If you encounter any error, please check the logfile that was displayed immediately after you started the script. This will contain all of the individual steps, commands used and the output of those commands.
LOGIN AND CHECK THE MIGRATED DATABASE
--> USE SQL*DEVELOPER TO CHECK IF THE PARKINGFINE USER HAS BEEN MIGRATED TO ATP
On your Desktop, you can see SQL*Developer. Start this application.
 
 CREATE A NEW CONNECTION TO ATP BY CLICKING ON THE GREEN + SIGN IN THE CONNECTIONS PANE
Connection Name	:	Choose your own name
Username	:	admin
Password	:	OraclePTS#2019 (or any other password you have used)
Connection Type	:	Cloud Wallet
Configuration File	:	<select the wallet you downloaded in /home/oracle/Downloads>
Service	:	<find YourService_tp>
 
 ENTER THE REQUIRED DETAILS AND PRESS CONNECT
After connecting, a new SQL Window will be displayed. Here you can execute queries on the ATP environment.
 
 ENTER THE QUERY AND PRESS THE GREEN ARROW TO EXECUTE IT
In the Query Result window, the result of the query will be displayed:
 
