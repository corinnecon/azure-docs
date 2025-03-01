---
title: Troubleshoot Azure Application Consistent Snapshot tool for Azure NetApp Files | Microsoft Docs
description: Provides troubleshooting content for using the Azure Application Consistent Snapshot tool that you can use with Azure NetApp Files. 
services: azure-netapp-files
documentationcenter: ''
author: Phil-Jensen
manager: ''
editor: ''

ms.assetid:
ms.service: azure-netapp-files
ms.workload: storage
ms.tgt_pltfrm: na
ms.topic: troubleshooting
ms.date: 05/17/2021
ms.author: phjensen
---

# Troubleshoot Azure Application Consistent Snapshot tool

This article provides troubleshooting content for using the Azure Application Consistent Snapshot tool that you can use with Azure NetApp Files and Azure Large Instance.

The following are common issues that you may encounter while running the commands. Follow the resolution instructions mentioned to fix the issue. If you still encounter an issue, open a Service Request from Azure portal and assign the request into the SAP HANA Large Instance queue for Microsoft Support to respond.

## Log files

One of the best sources of information for debugging any errors with AzAcSnap are the log files.  

### Log file location

The log files are stored in the directory configured per the `logPath` parameter in the AzAcSnap configuration file.  The default configuration filename is `azacsnap.json` and the default value for `logPath` is `"./logs"` which means the log files are written into the `./logs` directory relative to where the `azacsnap` command is run.  Making the `logPath` an absolute location (e.g. `/home/azacsnap/logs`) will ensure `azacsnap` outputs the logs into `/home/azacsnap/logs` irrespective of where the `azacsnap` command was run.

### Log file naming

The log filename is based on the application name (e.g. `azacsnap`), the  command option (`-c`) used (e.g. `backup`, `test`, `details`, etc.) and the configuration filename (e.g. default = `azacsnap.json`).  So if using the `-c backup` command, the log  filename by default would be `azacsnap-backup-azacsnap.log` and is written into the  directory configured by `logPath`.  

This naming convention was established to allow for multiple configuration files, one per database, and ensure ease of locating the associated logfiles.  Therefore, if the configuration filename is `SID.json`, then the result filename when using the `azacsnap -c backup --configfile SID.json` options will be `azacsnap-backup-SID.log`.

### Result file and syslog

For the `-c backup` command option AzAcSnap writes out to a `*.result` file and the system log (`/var/log/messages`) using the `logger` command.  The `*.result` filename has the same base name as the [log file](#log-file-naming) and goes into the same [location as the log file](#log-file-location).  It is a simple one line output file per the following examples.

Example output from `*.result` file.
```output
Database # 1 (PR1) : completed ok
```

Example output from `/var/log/messages` file.
```output
Dec 17 09:01:13 azacsnap-rhel azacsnap: Database # 1 (PR1) : completed ok
```

## Failed communication with Azure NetApp Files

When validating communication with Azure NetApp Files, communication might fail or time-out.  Check to ensure firewall rules are not blocking outbound traffic from the system running AzAcSnap to the following addresses and TCP/IP ports:-

- (https://)management.azure.com:443
- (https://)login.microsoftonline.com:443 

### Testing communication using Cloud Shell

You can test the Service Principal is configured correctly by using Cloud Shell through your Azure Portal. This will test that the configuration is correct bypassing network controls within a VNet or virtual machine. 

**Solution:**

1. Open a [Cloud Shell](../cloud-shell/overview.md) session in your Azure Portal. 
1. Make a test directory (e.g. `mkdir azacsnap`)
1. cd to the azacsnap directory and download the latest version of azacsnap tool.
    
    ```bash
    wget https://aka.ms/azacsnapinstaller
    ```
   
    ```output
    ----<snip>----
    HTTP request sent, awaiting response... 200 OK
    Length: 24402411 (23M) [application/octet-stream]
    Saving to: ‘azacsnapinstaller’

    azacsnapinstaller 100%[=================================================================================>] 23.27M 5.94MB/s in 5.3s

    2021-09-02 23:46:18 (4.40 MB/s) - ‘azacsnapinstaller’ saved [24402411/24402411]
    ```
    
1. Make the installer executable. (e.g. `chmod +x azacsnapinstaller`)
1. Extract the binary for testing.

    ```bash
    ./azacsnapinstaller -X -d .
    ```
    
    ```output
    +-----------------------------------------------------------+
    | Azure Application Consistent Snapshot Tool Installer |
    +-----------------------------------------------------------+
    |-> Installer version '5.0.2_Build_20210827.19086'
    |-> Extracting commands into ..
    |-> Cleaning up .NET extract dir
    ```

1. Using the Cloud Shell Upload/Download icon, upload the Service Principal file (e.g. `azureauth.json`) and the AzAcSnap configuration file for testing (e.g. `azacsnap.json`)
1. Run the Storage test from the Azure Cloud Shell console. 

    > [!NOTE]
    > The test command can take about 90 seconds to complete.

    ```bash
    ./azacsnap -c test --test storage
    ```

    ```output
    BEGIN : Test process started for 'storage'
    BEGIN : Storage test snapshots on 'data' volumes
    BEGIN : 1 task(s) to Test Snapshots for Storage Volume Type 'data'
    PASSED: Task#1/1 Storage test successful for Volume
    END : Storage tests complete
    END : Test process complete for 'storage'
    ```

## Problems with SAP HANA

### Running the test command fails

When validating communication with SAP HANA by running a test with `azacsnap -c test --test hana` and it provides the following error:

```output
> azacsnap -c test --test hana
BEGIN : Test process started for 'hana'
BEGIN : SAP HANA tests
CRITICAL: Command 'test' failed with error:
Cannot get SAP HANA version, exiting with error: 127
```

**Solution:**

1. Check the configuration file (for example, `azacsnap.json`) for each HANA instance to ensure the SAP HANA database values are correct.
1. Try to run the command below to verify if the `hdbsql` command is in the path and it can connect to the SAP HANA Server. The following example shows the correct running of the command and its output.

    ```bash
    hdbsql -n 172.18.18.50 - i 00 -d SYSTEMDB -U AZACSNAP "\s"
    ```

    ```output
    host          : 172.18.18.50
    sid           : H80
    dbname        : SYSTEMDB
    user          : AZACSNAP
    kernel version: 2.00.040.00.1553674765
    SQLDBC version:        libSQLDBCHDB 2.04.126.1551801496
    autocommit    : ON
    locale        : en_US.UTF-8
    input encoding: UTF8
    sql port      : saphana1:30013
    ```

    In this example, the `hdbsql` command isn't in the users `$PATH`.

    ```bash
    hdbsql -n 172.18.18.50 - i 00 -U AZACSNAP "select version from sys.m_database"
    ```

    ```output
    If 'hdbsql' is not a typo you can use command-not-found to lookup the package that contains it, like this:
    cnf hdbsql
    ```

    In this example, the `hdbsql` command is temporarily added to the user's `$PATH`, but when run shows the connection key hasn't been set up correctly with the `hdbuserstore Set` command (refer to Getting Started guide for details):

    ```bash
    export PATH=$PATH:/hana/shared/H80/exe/linuxx86_64/hdb/
    ```

    ```bash
    hdbsql -n 172.18.18.50 -i 00 -U AZACSNAP "select version from sys.m_database"
    ```

    ```output
    * -10104: Invalid value for KEY (AZACSNAP)
    ```

    > [!NOTE]
    > To permanently add to the user's `$PATH`, update the user's `$HOME/.profile` file

### Insufficient privilege

If running `azacsnap` presents an error such as `* 258: insufficient privilege`, check to ensure the appropriate privilege has been asssigned to the "AZACSNAP" database user (assuming this is the user created per the [installation guide](azacsnap-installation.md#enable-communication-with-database)).  Verify the user's current privilege with the following command:

```bash
hdbsql -U AZACSNAP "select GRANTEE,GRANTEE_TYPE,PRIVILEGE,IS_VALID,IS_GRANTABLE from sys.granted_privileges "' | grep -i -e GRANTEE -e azacsnap
```

```output
GRANTEE,GRANTEE_TYPE,PRIVILEGE,IS_VALID,IS_GRANTABLE
"AZACSNAP","USER","BACKUP ADMIN","TRUE","FALSE"
"AZACSNAP","USER","CATALOG READ","TRUE","FALSE"
"AZACSNAP","USER","CREATE ANY","TRUE","TRUE"
```

The error might also provide further information to help determine the required SAP HANA privileges, such as the output of `Detailed info for this error can be found with guid '99X9999X99X9999X99X99XX999XXX999' SQLSTATE: HY000`.  In this case follow SAP's instructions at [SAP Help Portal - GET_INSUFFICIENT_PRIVILEGE_ERROR_DETAILS](https://help.sap.com/viewer/b3ee5778bc2e4a089d3299b82ec762a7/2.0.05/en-US/9a73c4c017744288b8d6f3b9bc0db043.html) which recommends using the following SQL query to determine the detail on the required privilege.

```sql
CALL SYS.GET_INSUFFICIENT_PRIVILEGE_ERROR_DETAILS ('99X9999X99X9999X99X99XX999XXX999', ?)
```

```output
GUID,CREATE_TIME,CONNECTION_ID,SESSION_USER_NAME,CHECKED_USER_NAME,PRIVILEGE,IS_MISSING_ANALYTIC_PRIVILEGE,IS_MISSING_GRANT_OPTION,DATABASE_NAME,SCHEMA_NAME,OBJECT_NAME,OBJECT_TYPE
"99X9999X99X9999X99X99XX999XXX999","2021-01-01 01:00:00.180000000",120212,"AZACSNAP","AZACSNAP","DATABASE ADMIN or DATABASE BACKUP ADMIN","FALSE","FALSE","","","",""
```

In the example above, adding the 'DATABASE BACKUP ADMIN' privilege to the SYSTEMDB's AZACSNAP user, should resolve the insufficient privilege error.

### The `hdbuserstore` location

When setting up communication with SAP HANA, the `hdbuserstore` program is used to create the secure communication settings.  The `hdbuserstore` program is usually found under `/usr/sap/<SID>/SYS/exe/hdb/` or `/usr/sap/hdbclient`.  Normally the installer adds the correct location to the `azacsnap` user's `$PATH`.

## Failed test with storage

The command `azacsnap -c test --test storage` does not complete successfully.

### Azure Large Instance

The following example is from running `azacsnap` on SAP HANA on Azure Large Instance:

```bash
azacsnap -c test --test storage
```

```output
The authenticity of host '172.18.18.11 (172.18.18.11)' can't be established.
ECDSA key fingerprint is SHA256:QxamHRn3ZKbJAKnEimQpVVCknDSO9uB4c9Qd8komDec.
Are you sure you want to continue connecting (yes/no)?
```

**Solution:** The above error normally shows up when Azure Large Instance storage user has no access to the underlying storage. To validate access to storage with the provided storage user, run the `ssh`
command to validate communication with the storage platform.

```bash
ssh <StorageBackupname>@<Storage IP address> "volume show -fields volume"
```

An example with expected output:

```bash
ssh clt1h80backup@10.8.0.16 "volume show -fields volume"
```

```output
vserver volume
--------------------------------- ------------------------------
osa33-hana-c01v250-client25-nprod hana_data_h80_mnt00001_t020_vol
osa33-hana-c01v250-client25-nprod hana_data_h80_mnt00002_t020_vol
```

#### The authenticity of host '172.18.18.11 (172.18.18.11)' can't be established

```bash
azacsnap -c test --test storage
```

```output
BEGIN : Test process started for 'storage'
BEGIN : Storage test snapshots on 'data' volumes
BEGIN : 1 task(s) to Test Snapshots for Storage Volume Type 'data'
The authenticity of host '10.3.0.18 (10.3.0.18)' can't be established.
ECDSA key fingerprint is SHA256:cONAr0lpafb7gY4l31AdWTzM3s9LnKDtpMdPA+cxT7Y.
Are you sure you want to continue connecting (yes/no)?
```

**Solution:** Do not select Yes. Ensure that your storage IP address is correct. If there is still an
issue, confirm the storage IP address with Microsoft operations team.

### Azure NetApp Files

The following example is from running `azacsnap` on a VM using Azure NetApp Files:

```bash
azacsnap --configfile azacsnap.json.NOT-WORKING -c test --test storage
```

```output
BEGIN : Test process started for 'storage'
BEGIN : Storage test snapshots on 'data' volumes
BEGIN : 1 task(s) to Test Snapshots for Storage Volume Type 'data'
ERROR: Could not create StorageANF object [authFile = 'azureauth.json']
```

**Solution:**

1. Check for the existence of the Service Principal file, `azureauth.json`, as set in the `azacsnap.json` configuration file.
1. Check the log file (for example, `logs/azacsnap-test-azacsnap.log`) to see if the Service Principal (`azureauth.json`) has the correct content.  Example from log as follows:

      ```output
      [19/Nov/2020:18:39:49 +13:00] DEBUG: [PID:0020080:StorageANF:659] [1] Innerexception: Microsoft.IdentityModel.Clients.ActiveDirectory.AdalServiceException AADSTS7000215: Invalid client secret is provided.
      ```

1. Check the log file (for example, `logs/azacsnap-test-azacsnap.log`) to see if the Service Principal (`azureauth.json`) has expired. Example from log as follows:

      ```output
      [19/Nov/2020:18:41:10 +13:00] DEBUG: [PID:0020257:StorageANF:659] [1] Innerexception: Microsoft.IdentityModel.Clients.ActiveDirectory.AdalServiceException AADSTS7000222: The provided client secret keys are expired. Visit the Azure Portal to create new keys for your app, or consider using certificate credentials for added security: https://docs.microsoft.com/azure/active-directory/develop/active-directory-certificate-credentials
      ```

## Next steps

- [Tips](azacsnap-tips.md)