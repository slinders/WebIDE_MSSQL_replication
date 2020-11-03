# Set up real-time replication from source to SAP HANA in SAP Web IDE with MSSQL example

## Prerequisites
- A MSSQL database
- An SAP HANA database (this tutorial uses HANA Cloud)
- The SAP SDI Data Provisioning Agent (this tutorial uses version 2.5.1.2)
- Connection set up between HANA and the SDI Data Provisioning Agent
- A driver for MSSQL for SDI to connect (this tutorial uses mssql-jdbc-8.4.1.jre8.jar)

## Prep MSSQL database
For this tutorial, a Microsoft SQL Server Express Edition database was created using the 
AWS Relational Database Service. The database version is the latest available 14.* version 
and is sized as a db.t3.small.

### Configure MSSQL database
Using the admin user that is provided by AWS, a technical user is created which will later be used for HANA to logon with to this source database. 

```
USE master;
CREATE LOGIN HC_TECHNICAL_USER
 WITH PASSWORD = '<password>',
 CHECK_POLICY = OFF,
 CHECK_EXPIRATION = OFF;
```

A database is created
```
USE master; 
CREATE DATABASE hc;
```

The technical user is granted the required privileges according to the [SAP Help](https://help.sap.com/viewer/7952ef28a6914997abc01745fef1b607/2.0_SPS05/en-US/2815e1a621f84bada5fa3447d5029eb6.html).
The instructions are not all mandatory, there's a trade-ff between security concern and usability. I chose usability, meaning for example that I grant access to the entire database and not individual tables.
Also, *disclaimer*, this is one way you can configure your source database, but it might not be the best way for you. I'm assuming you know what you're doing.

In the new database, a new schema is created and for the technical user a user is created to login to this database

```
USE hc;
CREATE SCHEMA rep;
create user HC_TECHNICAL_USER for login HC_TECHNICAL_USER;
```
The user should also be allowed to create tables
```
--Allow the user to create tables
USE hc;
GRANT CREATE TABLE TO HC_TECHNICAL_USER;
```

It's up to you if you want to replicate DDL changes as well, I chose both DML/DDL.

```
--Creating a DML trigger requires ALTER permission on the table or schema on which the trigger is being created.
USE hc;
GRANT ALTER ON SCHEMA::rep TO HC_TECHNICAL_USER;

--Creating a DDL trigger with database scopes (ON DATABASE) requires ALTER ANY DATABASE DDL TRIGGER permission in the current database.
--Not needed if you don't want to replicate DDL changes
USE hc;
GRANT ALTER ANY DATABASE DDL TRIGGER TO HC_TECHNICAL_USER;

--GRANT CREATE PROCEDURE needed for SDI to create a generic procedure needed for replication
USE hc;
GRANT CREATE PROCEDURE TO HC_TECHNICAL_USER;

--GRANT VIEW SERVER STATE permission to view data processing state, such as transaction ID. This must be granted on the master database.
USE master;
GRANT VIEW SERVER STATE TO HC_TECHNICAL_USER;

```

Select access is needed to see data in the table, for example when viewing the content of the virtual table
```
USE hc;
GRANT SELECT ON SCHEMA::rep TO HC_TECHNICAL_USER;
```

Besides, we want to also use the technical user to insert, update or delete data so we can run some DML tests with that user
```
USE hc;
GRANT SELECT, INSERT, UPDATE, DELETE ON SCHEMA::rep TO HC_TECHNICAL_USER;

```

The following is (or was, if it's updated) not in the SDI documentation, but was needed to allow replication
```
--Create schema for SDI objects to manage replication. The schema should have the same name as the technical user name.
USE hc;
CREATE SCHEMA HC_TECHNICAL_USER;

--Grant privileges to technical user on the created schema
USE hc;
ALTER AUTHORIZATION ON SCHEMA::HC_TECHNICAL_USER TO HC_TECHNICAL_USER;
```

## Create source table and insert a few records of initial data
```
DROP TABLE REP.SALES;
CREATE TABLE REP.SALES (ID INTEGER, CREATION_DATE DATE, CUSTOMER_NAME NVARCHAR(100), PRODUCT_NAME NVARCHAR (100), QUANTITY INTEGER, PRICE DECIMAL, POS_COUNTRY NVARCHAR(100), PRIMARY KEY (ID));
INSERT INTO REP.SALES VALUES (1,'20200908','Cas Jensen','Toothbrush 747','6','261.54','United States of America');
INSERT INTO REP.SALES VALUES (2,'20201018','Barry French','Shampoo F100','2','199.99','Germany');
```

## Create remote source on HANA
The remote source can be set up using the Database Explorer, with a graphical UI. Below, the remote source is created using a create statement.
Make sure to replace:
- <host_name> with the hostname of your database
- <technical_user_password> with the password set for the HC_TECHNICAL_USER of the source database
- <agent_name> with the name of your agent 

```
DROP REMOTE SOURCE "RS_MSSQL" CASCADE;
CREATE REMOTE SOURCE "RS_MSSQL"
	ADAPTER "MssqlLogReaderAdapter" AT LOCATION AGENT "<agent_name>"
		CONFIGURATION '<?xml version="1.0" encoding="UTF-8"?>
<ConnectionProperties name="configurations">
  <PropertyGroup name="generic">
      <PropertyEntry name="pdb_dflt_column_repl">true</PropertyEntry>
  </PropertyGroup>
  <PropertyGroup name="data_type_conversion">
      <PropertyEntry name="map_char_types_to_unicode">false</PropertyEntry>
      <PropertyEntry name="allow_character_to_lob">true</PropertyEntry>
      <PropertyEntry name="map_time_to_timestamp">false</PropertyEntry>
  </PropertyGroup>
  <PropertyGroup name="database">
      <PropertyEntry name="pds_always_on">false</PropertyEntry>
      <PropertyEntry name="pds_server_name"><host_name>/PropertyEntry>
      <PropertyEntry name="pds_port_number">1433</PropertyEntry>
      <PropertyEntry name="pds_aglistener_host"></PropertyEntry>
      <PropertyEntry name="pds_aglistener_port"></PropertyEntry>
      <PropertyEntry name="pds_database_name">hc</PropertyEntry>
      <PropertyEntry name="pds_use_remote">false</PropertyEntry>
      <PropertyEntry name="additional_connection_properties"></PropertyEntry>
      <PropertyEntry name="remarksReporting">false</PropertyEntry>
      <PropertyEntry name="whitelist_table"></PropertyEntry>
  </PropertyGroup>
  <PropertyGroup name="schema_alias_replacements">
      <PropertyEntry name="schema_alias"></PropertyEntry>
      <PropertyEntry name="schema_alias_replacement"></PropertyEntry>
  </PropertyGroup>
  <PropertyGroup name="security">
      <PropertyEntry name="pds_use_ssl">false</PropertyEntry>
      <PropertyEntry name="pds_ssl_sc_cn"></PropertyEntry>
      <PropertyEntry name="pds_integrated_security">false</PropertyEntry>
      <PropertyEntry name="pds_use_agent_stored_credential">false</PropertyEntry>
  </PropertyGroup>
  <PropertyGroup name="cdc">
      <PropertyEntry name="capture_mode">trigger</PropertyEntry>
      <PropertyEntry name="pdb_dcmode">NATIVE</PropertyEntry>
      <PropertyEntry name="pds_ra_schema"></PropertyEntry>
      <PropertyEntry name="_lr_maint_id"></PropertyEntry>
      <PropertyEntry name="truncation_interval">10</PropertyEntry>
      <PropertyEntry name="skip_lr_errors">false</PropertyEntry>
      <PropertyEntry name="lr_max_op_queue_size">1000</PropertyEntry>
      <PropertyEntry name="lr_max_scan_queue_size">1000</PropertyEntry>
      <PropertyEntry name="scan_sleep_max">2</PropertyEntry>
      <PropertyEntry name="scan_sleep_increment">0</PropertyEntry>
      <PropertyEntry name="pds_sql_connection_pool_size">15</PropertyEntry>
      <PropertyEntry name="pds_retry_count">5</PropertyEntry>
      <PropertyEntry name="pds_retry_timeout">10</PropertyEntry>
      <PropertyEntry name="prefix">SDI_</PropertyEntry>
      <PropertyEntry name="enable_manageable_trigger_prefix">false</PropertyEntry>
      <PropertyEntry name="manageable_trigger_prefix">/1DI/</PropertyEntry>
      <PropertyEntry name="conn_pool_size">8</PropertyEntry>
      <PropertyEntry name="min_scan_interval">0</PropertyEntry>
      <PropertyEntry name="max_scan_interval">1</PropertyEntry>
      <PropertyEntry name="max_batch_size">128</PropertyEntry>
      <PropertyEntry name="batch_queue_size">64</PropertyEntry>
      <PropertyEntry name="max_transaction_count">1000</PropertyEntry>
      <PropertyEntry name="max_scan_size">50000</PropertyEntry>
      <PropertyEntry name="record_pk_only">false</PropertyEntry>
      <PropertyEntry name="capture_before_after_images">false</PropertyEntry>
  </PropertyGroup>
</ConnectionProperties>
'
			WITH CREDENTIAL TYPE 'PASSWORD'
USING '<CredentialEntry name="credential"><user>HC_TECHNICAL_USER</user>
<password><technical_user_password></password></CredentialEntry>';
```

## Test replication with SQL statements
Before building the Web IDE project with replication tasks, it might be a good idea to first check if replication works on the HANA level. 
This can be done with the following statements. The assumption is that you are using a user with the required privileges on the remote source to create subscriptions.
You could also do these tests using the graphical UI of the DB Explorer.

```
CREATE SCHEMA REP_MSSQL;

-- setup subscription
CREATE VIRTUAL TABLE "REP_MSSQL"."V_SALES" at "RS_MSSQL"."<NULL>"."rep"."SALES";
CREATE TABLE "REP_MSSQL"."T_SALES" LIKE "REP_MSSQL"."V_SALES";
CREATE REMOTE SUBSCRIPTION "REP_MSSQL"."SUB_SALES" ON "REP_MSSQL"."V_SALES" TARGET TABLE "REP_MSSQL"."T_SALES" ;
ALTER REMOTE SUBSCRIPTION "REP_MSSQL"."SUB_SALES" QUEUE;

-- do initial load before distribute remote subscription
INSERT INTO "REP_MSSQL"."T_SALES"  (SELECT * FROM "REP_MSSQL"."V_SALES");
ALTER REMOTE SUBSCRIPTION "REP_MSSQL"."SUB_SALES" DISTRIBUTE;
-- the following should return the two records inserted earlier into the source table
SELECT * FROM "REP_MSSQL"."T_SALES";
-- execute the following on the source table in mssql
INSERT INTO REP.SALES VALUES (3,'20201020','Zeph Skater','Helmet C172','30','300.00','Spain');
-- check if the record is replicated to the target table
SELECT * FROM "REP_MSSQL"."T_SALES";
```

If the SQL tests are successful, the created objects can again be removed with the following statements

```
-- stop the replication
ALTER REMOTE SUBSCRIPTION "REP_MSSQL"."SUB_SALES" RESET;
DROP REMOTE SUBSCRIPTION "REP_MSSQL"."SUB_SALES";
DROP TABLE "REP_MSSQL"."V_SALES";
DROP TABLE "REP_MSSQL"."T_SALES";
```

## Web IDE project with design time Replication Tasks
If you are happy with building your replications with solely SQL, this would be your endpoint. 
But now the fun really starts with the SAP Web IDE, where a project will be created with design time files,
replicating the same (and more) as was done earlier with SQL statements.

### Create a user in HANA for the User Provided Service
A User Provided Service will be created later, but first we need to create a user on the HANA database that this service can leverage.
The user that we create should have the privilege *to grant access* to create virtual tables, create remote subscriptions, and to process remote subscriptions.

```
DROP USER cups_mssql_remote_source;
CREATE USER cups_mssql_remote_source PASSWORD "<password>" NO FORCE_FIRST_PASSWORD_CHANGE;
ALTER USER cups_mssql_remote_source DISABLE PASSWORD LIFETIME;
GRANT CREATE VIRTUAL TABLE ON REMOTE SOURCE "RS_MSSQL" TO cups_mssql_remote_source WITH GRANT OPTION; --needed for object owner and reptask editor to browse remote source
GRANT ALTER ON REMOTE SOURCE "RS_MSSQL" TO cups_mssql_remote_source WITH GRANT OPTION; --needed for reptask editor to browse remote source
GRANT CREATE REMOTE SUBSCRIPTION ON REMOTE SOURCE "RS_MSSQL" TO cups_mssql_remote_source WITH GRANT OPTION;
GRANT PROCESS REMOTE SUBSCRIPTION EXCEPTION ON REMOTE SOURCE "RS_MSSQL" TO cups_mssql_remote_source WITH GRANT OPTION;
```

### Create a User Provided Service
Creating a user provided service can also be done from the Web IDE. Below the definition is displayed, which can also be used to create the service from the command line or the SCP Admin UI.
When creating the service, make sure to include the certificate as described in the [SAP Help](https://help.sap.com/viewer/f1b440ded6144a54ada97ff95dac7adf/2.6/en-US/c8d62da2150c4b1ebb8f9460d2cfde74.html)
Make sure to put in the password and the right host and API endpoint of the HANA database. 
If you are configuring an on-prem HANA database, you can leave out the endpoint, encrypt and certificate tags.
```
{
    "host": "<database-id>.hana.prod-eu20.hanacloud.ondemand.com",
    "port": "443",
    "user": "cups_mssql_remote_source",
    "password": "<password>",
    "driver": "com.sap.db.jdbc.Driver",
    "tags": [
        "hana"
    ],
    "endpoint": "https://api.cf.eu20.hana.ondemand.com",
    "encrypt": true,
    "certificate": "-----BEGIN CERTIFICATE-----\nMIIDrzCCApegAwIBAgIQCDvgVpBCRrGhdWrJWZHHSjANBgkqhkiG9w0BAQUFADBh\nMQswCQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMRkwFwYDVQQLExB3\nd3cuZGlnaWNlcnQuY29tMSAwHgYDVQQDExdEaWdpQ2VydCBHbG9iYWwgUm9vdCBD\nQTAeFw0wNjExMTAwMDAwMDBaFw0zMTExMTAwMDAwMDBaMGExCzAJBgNVBAYTAlVT\nMRUwEwYDVQQKEwxEaWdpQ2VydCBJbmMxGTAXBgNVBAsTEHd3dy5kaWdpY2VydC5j\nb20xIDAeBgNVBAMTF0RpZ2lDZXJ0IEdsb2JhbCBSb290IENBMIIBIjANBgkqhkiG\n9w0BAQEFAAOCAQ8AMIIBCgKCAQEA4jvhEXLeqKTTo1eqUKKPC3eQyaKl7hLOllsB\nCSDMAZOnTjC3U/dDxGkAV53ijSLdhwZAAIEJzs4bg7/fzTtxRuLWZscFs3YnFo97\nnh6Vfe63SKMI2tavegw5BmV/Sl0fvBf4q77uKNd0f3p4mVmFaG5cIzJLv07A6Fpt\n43C/dxC//AH2hdmoRBBYMql1GNXRor5H4idq9Joz+EkIYIvUX7Q6hL+hqkpMfT7P\nT19sdl6gSzeRntwi5m3OFBqOasv+zbMUZBfHWymeMr/y7vrTC0LUq7dBMtoM1O/4\ngdW7jVg/tRvoSSiicNoxBN33shbyTApOB6jtSj1etX+jkMOvJwIDAQABo2MwYTAO\nBgNVHQ8BAf8EBAMCAYYwDwYDVR0TAQH/BAUwAwEB/zAdBgNVHQ4EFgQUA95QNVbR\nTLtm8KPiGxvDl7I90VUwHwYDVR0jBBgwFoAUA95QNVbRTLtm8KPiGxvDl7I90VUw\nDQYJKoZIhvcNAQEFBQADggEBAMucN6pIExIK+t1EnE9SsPTfrgT1eXkIoyQY/Esr\nhMAtudXH/vTBH1jLuG2cenTnmCmrEbXjcKChzUyImZOMkXDiqw8cvpOp/2PV5Adg\n06O/nVsJ8dWO41P0jmP6P6fbtGbfYmbW0W5BjfIttep3Sp+dWOIrWcBAI+0tKIJF\nPnlUkiaY4IBIqDfv8NZ5YBberOgOzW6sRBc4L0na4UU+Krk2U886UAb3LujEV0ls\nYSEY1QSteDwsOoBrp+uvFRTp2InBuThs4pFsiv9kuXclVzDAGySj4dzp30d8tbQk\nCAUw7C29C79Fv1C5qfPrmAESrciIxpg0X40KPMbp1ZWVbd4=\n-----END CERTIFICATE-----"
}
```

Now the config is complete and the Web IDE project can now be build.

## Reference
Hdbgrants syntax for remote remote_sources: [SAP Help](https://help.sap.com/viewer/4505d0bdaf4948449b7f7379d24d0f0d/2.0.03/en-US/f49c1f5c72ee453788bf79f113d83bf9.html)

SQL statements for test data and subscription role assignment
```
DROP TABLE REP.SALES;
CREATE TABLE REP.SALES (ID INTEGER, CREATION_DATE DATE, CUSTOMER_NAME NVARCHAR(100), PRODUCT_NAME NVARCHAR (100), QUANTITY INTEGER, PRICE DECIMAL, POS_COUNTRY NVARCHAR(100), PRIMARY KEY (ID));
INSERT INTO REP.SALES VALUES (1,'20200908','Cas Jensen','Toothbrush 747','6','261.54','United States of America');
INSERT INTO REP.SALES VALUES (2,'20201018','Barry French','Shampoo F100','2','199.99','Germany');
INSERT INTO REP.SALES VALUES (3,'20201020','Zeph Skater','Helmet C172','30','300.00','Spain');
INSERT INTO REP.SALES VALUES (4,'20201102','Clay Rozendal','Longboard A380','30','4965.76','Luxembourg');
DELETE FROM REP.SALES WHERE ID=4;
INSERT INTO REP.SALES VALUES (4,'20201102','Clay Rozendal','Shortboard T20','10','20.0','Luxembourg');
UPDATE REP.SALES SET QUANTITY=50 WHERE ID=4;

CREATE USER TEST_MSSQL_SUBS PASSWORD "<PASSWORD>" NO FORCE_FIRST_PASSWORD_CHANGE;
REVOKE MSSQL_REPLICATION_1."manage_all_subscriptions" FROM TEST_MSSQL_SUBS;
GRANT MSSQL_REPLICATION_1."manage_all_subscriptions" TO TEST_MSSQL_SUBS;
REVOKE MSSQL_REPLICATION_1."manage_specific_subscriptions" FROM TEST_MSSQL_SUBS;
GRANT MSSQL_REPLICATION_1."manage_specific_subscriptions" TO TEST_MSSQL_SUBS;

--below privilege cannot be part of *.hdbrole, so has to be done runtime
GRANT PROCESS REMOTE SUBSCRIPTION EXCEPTION ON REMOTE SOURCE "RS_MSSQL" TO TEST_MSSQL_SUBS;
```