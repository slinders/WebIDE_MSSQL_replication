# Set up real-time replication from source to SAP HANA in SAP Web IDE with MSSQL example

## Prep MSSQL database
For this tutorial, a Microsoft SQL Server Express Edition database was created using the 
AWS Relational Database Service. The database version is the latest available 14.* version 
and it runs on db.t3.small
 
Created user and test objects

USE master;
CREATE LOGIN DWC_TECHNICAL_USER
 WITH PASSWORD = '<password>',
 CHECK_POLICY = OFF,
 CHECK_EXPIRATION = OFF;

GRANT VIEW SERVER STATE TO DWC_TECHNICAL_USER;

-- use the master database to create your new database:

USE master;  
CREATE DATABASE test;

-- then, use your new database to create your schema:

USE test;
GRANT CONTROL ON DATABASE::test TO DWC_TECHNICAL_USER;
CREATE SCHEMA test_schema;
create user DWC_TECHNICAL_USER for login DWC_TECHNICAL_USER;
grant select on schema :: test_schema to DWC_TECHNICAL_USER;

-- create table
create table T1 (a integer, b integer);
insert into T1 values (0,1); 

-- check and change password
ALTER LOGIN DWC_TECHNICAL_USER WITH PASSWORD = '<password>',
 CHECK_POLICY = OFF,
 CHECK_EXPIRATION = OFF;

EXEC sp_readerrorlog 0, 1, 'Login failed' 

--using DWC_TECHNICAL_USER to get over error that this schema could not be found
create schema admin;

DROP USER AGENT_ADMIN;
CREATE USER AGENT_ADMIN PASSWORD "<password>" NO FORCE_FIRST_PASSWORD_CHANGE SET USERGROUP DEFAULT;
ALTER USER AGENT_ADMIN DISABLE PASSWORD LIFETIME;
grant agent admin to AGENT_ADMIN;
grant adapter admin to AGENT_ADMIN;
CREATE USER AGENT_MESSENGER PASSWORD "<password>" NO FORCE_FIRST_PASSWORD_CHANGE SET USERGROUP DEFAULT;
ALTER USER AGENT_MESSENGER DISABLE PASSWORD LIFETIME;

CREATE SCHEMA REP_MSSQL;

--create remote source
CREATE REMOTE SOURCE "aws_fra_mssql"
	ADAPTER "MssqlLogReaderAdapter"
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
      <PropertyEntry name="pds_server_name">something.rds.amazonaws.com</PropertyEntry>
      <PropertyEntry name="pds_port_number">1433</PropertyEntry>
      <PropertyEntry name="pds_aglistener_host"></PropertyEntry>
      <PropertyEntry name="pds_aglistener_port"></PropertyEntry>
      <PropertyEntry name="pds_database_name">test</PropertyEntry>
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
USING '<CredentialEntry name="credential"><user></user>
<password></password></CredentialEntry>';

-- setup subscription
CREATE VIRTUAL TABLE "REP_MSSQL"."V_SALES" at "aws_fra_mssql"."<NULL>"."reptest"."SALES";
CREATE TABLE "REP_MSSQL"."T_SALES" LIKE "REP_MSSQL"."V_SALES";
CREATE REMOTE SUBSCRIPTION "REP_MSSQL"."SUB_SALES" ON "REP_MSSQL"."V_SALES" TARGET TABLE "REP_MSSQL"."T_SALES" ;
ALTER REMOTE SUBSCRIPTION "REP_MSSQL"."SUB_SALES" QUEUE;

-- do initial load before distribute remote subscription
INSERT INTO "REP_MSSQL"."T_SALES"  (SELECT * FROM "REP_MSSQL"."V_SALES");
ALTER REMOTE SUBSCRIPTION "REP_MSSQL"."SUB_SALES" DISTRIBUTE;
-- do some dmls in REP_MSSQL and check change data here
SELECT * FROM "REP_MSSQL"."T_SALES";

-- stop the replication
ALTER REMOTE SUBSCRIPTION "REP_MSSQL"."SUB_SALES" RESET;
DROP REMOTE SUBSCRIPTION "REP_MSSQL"."SUB_SALES";
DROP TABLE "REP_MSSQL"."V_SALES";
DROP TABLE "REP_MSSQL"."T_SALES";

--service grantor user
--when creating the service, make sure to include the certificate: https://help.sap.com/viewer/f1b440ded6144a54ada97ff95dac7adf/2.6/en-US/c8d62da2150c4b1ebb8f9460d2cfde74.html

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

CREATE USER cups_mssql_remote_source PASSWORD "<password>" NO FORCE_FIRST_PASSWORD_CHANGE;
ALTER USER cups_mssql_remote_source DISABLE PASSWORD LIFETIME;
GRANT CREATE VIRTUAL TABLE ON REMOTE SOURCE "aws_fra_mssql" TO cups_mssql_remote_source WITH GRANT OPTION;
GRANT CREATE REMOTE SUBSCRIPTION ON REMOTE SOURCE "aws_fra_mssql" TO cups_mssql_remote_source WITH GRANT OPTION;
GRANT PROCESS REMOTE SUBSCRIPTION EXCEPTION ON REMOTE SOURCE "aws_fra_mssql" TO cups_mssql_remote_source WITH GRANT OPTION;
GRANT ALTER ON REMOTE SOURCE "aws_fra_mssql" TO cups_mssql_remote_source WITH GRANT OPTION;

--hdbgrants syntax for remote remote_sources
https://help.sap.com/viewer/4505d0bdaf4948449b7f7379d24d0f0d/2.0.03/en-US/f49c1f5c72ee453788bf79f113d83bf9.html
