# Upgrade to 19c Autonomous Database #

In this lab will be show the usage of a new tool called MV2ADB. This tool can, after completing the configuration file, execute all the steps to export, transport and import a database to the Oracle Autonomous Cloud.

## Disclaimer ##
The following is intended to outline our general product direction. It is intended for information purposes only, and may not be incorporated into any contract. It is not a commitment to deliver any material, code, or functionality, and should not be relied upon in making purchasing decisions. The development, release, and timing of any features or functionality described for Oracle’s products remains at the sole discretion of Oracle.

## Lab contents ##

## Download and install the required tools ##

The script that allows you easy migration to ADB can be downloaded from MyOracle Support through note **2463574.1**. In the Workshop environment we have already downloaded the tool for you in the `/source` directory.

### Install the MV2ADB script ###
The MV2ADB tool is an .rpm package and needs to be installed as the root user.

````
[oracle@ws ~]$ <copy>sudo yum -y localinstall /source/mv2adb*.rpm</copy>
````
You should see a similar output:

````
Loaded plugins: langpacks, ulninfo
Examining /source/mv2adb-2.0.1-80.noarch.rpm: mv2adb-2.0.1-80.noarch
Marking /source/mv2adb-2.0.1-80.noarch.rpm to be installed
Resolving Dependencies
--> Running transaction check
---> Package mv2adb.noarch 0:2.0.1-80 will be installed
--> Finished Dependency Resolution

==========================================================================================
 Package            Arch            Version        Repository                       Size
==========================================================================================
Installing:
 mv2adb             noarch          2.0.1-80       /mv2adb-2.0.1-80.noarch          273 k

Transaction Summary
==========================================================================================
Install  1 Package

Total size: 273 k
Installed size: 273 k
Downloading packages:
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : mv2adb-2.0.1-80.noarch                                                 1/1

MV2ADB has been installed on /opt/mv2adb succesfully!

  Verifying  : mv2adb-2.0.1-80.noarch                                                 1/1

Installed:
  mv2adb.noarch 0:2.0.1-80

Complete!
````

Please note that the install script shows the location where the tool has been installed. In this case **`/opt/mv2adb`**. We need this later in the Lab.

### Check already installed Oracle Instant Client ###

We have already downloaded and unzipped the required files for the Oracle Instant Client. In the directory `/opt/instantclient` we have unzipped the base Instant Client, the SQL*Plus zip file and the Tools zip file. All have been downloaded from OTN.

Make sure the directory 19.6 exists and that the directory has contents by executing the following command:

````
$ <copy>ls -l /opt/instantclient/19.6/</copy>
````

Output similar to the following should be visible:

    [oracle@upgradews-4 ~]$ ls -l /opt/instantclient/19.6/
    total 235040
    -rwxr-xr-x 1 oracle oinstall 40617 Jun 28 2018 adrci
    -rw-r--r-- 1 oracle oinstall 1317 Jun 28 2018 BASIC_README
    -rwxr-xr-x 1 oracle oinstall 1066102 Jun 28 2018 exp
    -rwxr-xr-x 1 oracle oinstall 228672 Jun 28 2018 expdp
    -rwxr-xr-x 1 oracle oinstall 57556 Jun 28 2018 genezi
    ...

## Create a new Autonomous Database ##

Since we are migrating to Oracle Autonomous Database, we will need to create a new ATP database before we can continue. 

### Login to Oracle Cloud Infrastructure ###

In your Workshop hand-out you will see an Oracle Cloud Infrastructure URL like `https://console.<region>.oraclecloud.com`. Locate this URL and enter it into the browser inside the supplied image. In the same section on the handout, you will see the Cloud Tenancy, the username and the password.

Execute the following steps:

- Navigate to the Oracle Cloud Infrastructure Console
	- Enter the Tenancy name and submit
	- Enter the username and password supplied and press submit

When logged in, execute the following steps

- Use the left side menu (stacked menu or hamburger menu)
- Navigate to the Autonomous Transaction Processing menu

In this menu, you can see the databases currently running and you can create new databases. 

### Select correct region and compartment ###

Since Oracle has several regions in the world to host the databases and your administrator might have restricted the locations where you can create those databases, make sure you have selected the correct values for the following items:

- Datacenter Region
	- Like US West (Phoenix) or Germany Central (Frankfurt)
	- Correct region can be found on your hand-out
- Compartment
	- By default the (root) compartment is selected
	- Select the Compartment you have access to
	- Correct compartment can be found on your hand-out
	- Compartment name in PTS workshops has the format `ADB-COMPARTMENT-<city>-<date>`
 
### Create a new Autonomous Transaction Processing database ###

By clicking on the blue 'Create Autonomous Database' button, the wizard will display that helps you with the creation. Enter the following values for your new database:

    Workload Type : Autonomous Transaction Processing
    Compartment	  : <keep value>
    Display Name  : ATP-<your-name>
    Database Name : ATP<your-initials>
    CPU Core Count: 1
    Storage (TB)  : 1
    Autoscaling   : unchecked
    Password      : OraclePTS#2019
    License Type  : BYOL or 'My Organization Already owns Oracle Database (etc..)'

After this, click on the **'Create Autonomous Database'** button to start the process. This process should not take more than a few minutes to complete. 

You may continue with the Lab guide while the database is being created.

## Create an Object Store Bucket ##

The MV2ADB tool needs to upload one or several Datapump dump files to the Oracle Cloud Environment before the import process can start. It requires Cloud Object storage to do this.

The MV2ADB tool can create the required Object Store Bucket as part of the execution of the script but this requires the install and setup of the OCI Command Line Interface. To bypass the extra install requirement, we will pre-create the bucket ourselves in the OCI Console.

### Navigate to the Object Storage menu in the OCI Console ###

Execute the following steps:

- Make sure you are logged into the OCI Cloud console
- Use the left side menu (stacked menu or hamburger menu)
- Navigate to the Object Storage menu
- Make sure you are working in the correct region
- Make sure you have selected the correct compartment

To create a new bucket (which behaves similar to a subdirectory in a filesystem), click on the blue **'Create Bucket'** button. A 'Create Bucket' window will be displayed where you can enter the Bucket name.

- Do not use the default name
- Enter a unique name that you can remember
	- Like 'Upgrade-<Your Initials>'
- Make sure you store the bucket name somewhere
	- Copy/paste it on a notepad or piece of papier
	- The name of the bucket is case sensitive
	- We need this name in the configuration file for MV2ADB

## Gather parameters and create the MV2ADB config file ##

The MV2ADB script needs input parameters so that it can identify the source database, the schemas to be migrated and the destination system. In the following sections, we will go through the minimum required parameters needed for this lab and the migration. 

For additional parameters, please check the MV2ADB documentation in the MOS note.

### Create the initial configuration file ###

In this section will we create an initial configuration file. Some parameters have already been filled with values, some values you will need to gather and enter. We will create an empty config file using a text editor. 

Text editing on the supplied Linux image can be done using any tool you know in the image like `vi` (for people who know how vi works) or for example `gedit` which works similar to Notepad in a Windows environment. In all examples in this Lab where you see `vi` used, you can replace `vi` by `gedit`.

If you are currently not logged in as the root user, execute the following command:
````
$ <copy>sudo -s</copy>
````

Create the new configuration file by executing the following command:
````
# <copy>vi /opt/mv2adb/conf/atp.mv2adb.cfg</copy>
````

Cut-and-paste the following default setup file in the new config file:
````
<copy>
# DB Parameters
DB_CONSTRIG=//localhost:1521/DB112
SYSTEM_DB_PASSWORD=53152A9726C00647158CD4B1E103F1F2
SCHEMAS=PARKINGFINE
DUMPFILES=/tmp/DB112-ATP.dmp
OHOME=/u01/app/oracle/product/11.2.0/dbhome_112
ICHOME=/opt/instantclient/19.6

# Expdp/Impdp Parameters
ENC_PASSWORD=53152A9726C00647158CD4B1E103F1F2
ENC_TYPE=AES256

# Object Store Properties
BMC_HOST=
BMC_TENNANT=
BMC_BUCKET=
BMC_ID=
BMC_PASSWORD=

# ADB Parameters
ADB_NAME=
ADB_PASSWORD=
CFILE=
</copy>
````

In the next sections we will locate the required parameters and put them in this config file. Please keep this config file open but save it on a regular basis so that no information is lost if something goes wrong. If you need to access the Operating System, please open a second terminal window.

### Gather the (Source) DB Parameters ###

The initial section is regarding the source database. Since this is a Lab environment, we know what the connection details are and which schema we want to migrate. In a regular setup, you need to change the options to your settings.

Here is some information where you can find the details:

- **DB_CONSTRIG**
	- Connecting string from the local system (where mv2adb is running) to the database instance that needs to be migrated
- **SYSTEM_DB_PASSWORD**
	- Password for SYSTEM user for this source database
- **SCHEMAS**
	- Schema's to be exported
	- Only transport the schema's that you need, do not include any default schema's like HR, OE, SYSTEM, SYS etc as they already exist in ADB and might result in errors (or get ignored)
- **DUMPFILES**
	- File system location of dumpfiles. 
	- If you want parallelism during export and import, specify as many files as you want the parallelism to be.
	- Make sure the files are unique in the source but also in the destination (ATP) environment.
- **OHOME**
	- The source database Oracle Home
- **IHOME**
	- The installed Oracle Instant Client home (basic, SQL*Plus and Tools unzipped)

### Gathering Expdb/Impdb parameters ###

The MV2ADB tool only allows you to transport encrypted dumpfiles to the OCI Cloud to make sure your data cannot be compromised when exporting and uploading your data. In this section of the config file, you need to specify the password and the encryption type. We have already entered a password and encryption type for you.

- **ENC_PASSWORD**
	- A password that will encrypt your datapump exports. 
	- It has nothing to do with any existing user or password. 
	- Please note that this password cannot be plain text, it needs to be encrypted
- **END_TYPE**
	- Type of encryption of your data. 
	- The higher the number, the more encryption but also slower export/import. 
	- Options are  **AES128**, **AES192** and **AES256**

In case you want to specify your own password, you need will need to encrypt this password using the MV2ADB tooling. For this, the command `encpwd` has been created.

Example: If your encryption password is 'Welcome_123', you would need to execute the following command:

````
# <copy>/opt/mv2adb/mv2adb encpass</copy>
````

After this, you enter the password your want to encrypt (twice) and the encrypted value will be displayed:

    Please enter the password :
    Please re-enter the password :
    53152A9726C00647158CD4B1E103F1F2

### Gathering Object Store properties ###

The MV2ADB script can currently only work with .dmp dumpfiles using the Swift compatible interface and Swift compatible storage. The parameters in the 'Object Store Properties' section specify where the dumpfiles should be uploaded to after the export and imported from during the import. This location is also where the logfiles of the impdp process will be stored.

If the OCI Command Line Interface (OCI CLI) is installed on the MV2ADB server, you do not need to enter the Swift compatible details. See the example config in the documentation for more information.

- **BMC_HOST**
	- This is the Swift object storage URL for your environment. 
	- It usually has the form of `https://swiftobjectstorage.<region>.oraclecloud.com` 
	- Region is **`us-phoenix-1`**, **`eu-frankfurt-1`** or similar. Please check your hand-out for details
- **BMC_TENNANT**
	- Name of your Tenancy. 
	- Be aware, for SWIFT we only use lower case naming
- **BMC_BUCKET**
	- The name of an object storage bucket that will be used for this migration. 
	- In the previous section you pre-created a bucket with a unique name.
- **BMC_ID**
	- Your username in OCI
	- Example: ADB-<city>-<date>
- **BMC_PASSWORD**
	- This is NOT the console password for your user !
	- The Swift interface needs a so-called Authentication Token
	- This Auth Token is user specific and would normally be generated by you
	- In this workshop, we have already generated the Auth Token for you.

Before we can enter all information into our configuration file, we need to encrypt the Swift Authentication Token using the mv2adb password encoder. The clear-text value of the generated Auth token can be found in the Object storage bucket in your compartment. Locate the file in the **`Lab-Artifacts-<xxx>`** bucket.

- Navigate to the Object Storage section of OCI
- Click on your Lab-Artifacts-<xx> bucket
- Locate the file called auth_token.txt
- Click on the stacked menu for the file and choose 'Object Details'
	- The Authentication Token will be displayed and consists of 20 random characters.

We can now encrypt the Authentication Token by running the mv2adb password encoder.

````
# <copy>/opt/mv2adb/mv2adb encpass</copy>
````

- Select the 20 characters of the Authentication Token in your console screen
	- Right click your mouse and select Copy
- Navigate to the terminal window where the `mv2adb` script is waiting for input
	- Paste the copied value twice (and press enter if needed after every paste action)

````
Please enter the password : <cut-and-paste auth_key>
Please re-enter the password : <cut-and-paste auth_key>
765B5FDA9F3F6110E7AB9E7D808612B3E38DA130FBCC2E3C0535DA923DF46CDF
````

Now that we have the last value we need for the Object Store settings, please enter the required information into the Object Store Properties section of the configuration file. Below is an **example** of the configuration file.

**Please do not copy these values, use your own values to complete YOUR file**

````
# Object Store Properties 
BMC_HOST=https://swiftobjectstorage.eu-frankfurt-1.oraclecloud.com
BMC_TENNANT=oraclepartnersas
BMC_BUCKET=ATP-MyInitials
BMC_ID=ADB-<city>-<date>
BMC_PASSWORD= 765B5FDA9F3F6110E7AB9E7D....12B3E38DA130FBCC2E3C0535DA923DF46CDF
````

### Gathering the ADB Parameters ###

During the gathering of the other parameters, your ADB environment should have been created. As a last step we will now gather the information needed for the last section in your configuration file:

- **ADB_NAME**
	- Name of your ADB instance
	- This is the **Database Name** and NOT the **Display Name** 
- **ADB_PASSWORD**
	- Database ADMIN password
	- Encrypted using the mv2adb tool
- **CFILE**
	- Zipfile containing database credentials
	 
**First parameter** requires the name of your created ADB environment. 
- Navigate to the ADB Console and find the name of your database. 
- This should be the 'Database Name' value and not the 'Display Name' value
- Enter the Database Name value into the configuration file
 
**Second parameter** is the password you have entered while creating the Autonomous environment. 
- If you have used the suggested password, it would be OraclePTS#2019. 
	- If you have chosen another password, you now need to remember it
- Encrypt the database password using the `/opt/mv2adb/mv2adb encpass` command
- Enter the result innto your configuration file

**Third parameter** requires a download from the OCI console for this Autonomous Database, the so-called Wallet file.
- Navigate to the Autonomous Database you have created
- Click on the Display name of the database and display the details for the instance
- Click on the DB Connection button next to the green 'ATP' image
- Select Wallet type = Instance Wallet
	- Click on Download Wallet
	- In the following screen a password is requested. 
	- This is the password that protects the keystore inside the zipfile. 
	- For this exercise we will not be using this keystore so enter **any random password** twice.
	- Your zipfile will be downloaded to the default location `/home/oracle/Downloads`. 
- Please note the exact name and location of the wallet.zip and enter this in your parameters.

Example values for the ADB Parameters section:

**Please do not copy these values, use your own values to complete YOUR file**

````
# ADB Parameters 
ADB_NAME=ATPINIT
ADB_PASSWORD= DE3D105A8E6F6A4D5E8XXXSW6BC1D3BA
CFILE=/home/oracle/Downloads/Wallet_ATPINIT.zip
````

#### Make sure all parameters have a value and save the file ####

The file should be in the `/opt/mv2adb/conf` directory and is called `ATP.mv2adb.conf`

````
# <copy>ls -l /opt/mv2adb/conf</copy>
````

````
total 12
-rw-r--r-- 1 root root  471 May 17 13:13 ATP.mv2adb.conf
-rwxr-xr-x 1 root root 5159 Dec 28 12:52 DBNAME.mv2adb.cfg
````

## Check Source schema(s) for compatibility ##

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
    ========================================================================    ==============
    --------------------------------------------------------------------------------------
    -- ATP MIGRATION SUMMARY
    --------------------------------------------------------------------------------------
                                                               Objects             Total
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
````
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
````




START THE MIGRATION USING THE MV2ADB SCRIPT
Now we can start the actual migration by starting the MV2ADB tool using the proper options.
 START THE MV2ADB SCRIPT USING THE CONFIGURATIONFILE YOU JUST CREATED
[oracle@ws ~]$ sudo /opt/mv2adb/mv2adb auto -conf /opt/mv2adb/conf/ATP.mv2adb.conf

````
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
````
After about 10 minutes, all the steps should have been executed successfully. If you encounter any error, please check the logfile that was displayed immediately after you started the script. This will contain all of the individual steps, commands used and the output of those commands.
LOGIN AND CHECK THE MIGRATED DATABASE
USE SQL*DEVELOPER TO CHECK IF THE PARKINGFINE USER HAS BEEN MIGRATED TO ATP
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
 
