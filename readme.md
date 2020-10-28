--prepping MSSQL database
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

-- setup subscription
CREATE VIRTUAL TABLE "REP_MSSQL"."V_T2_PK" at "aws_fra_mssql"."<NULL>"."reptest"."T2_PK";
CREATE TABLE "REP_MSSQL"."T_T2_PK" LIKE "REP_MSSQL"."V_T2_PK";
CREATE REMOTE SUBSCRIPTION "REP_MSSQL"."SUB_T2_PK" ON "REP_MSSQL"."V_T2_PK" TARGET TABLE "REP_MSSQL"."T_T2_PK" ;
ALTER REMOTE SUBSCRIPTION "REP_MSSQL"."SUB_T2_PK" QUEUE;

-- do initial load before distribute remote subscription
INSERT INTO "REP_MSSQL"."DPSIMPLE"  (SELECT * FROM "REP_MSSQL"."V_T2_PK");
ALTER REMOTE SUBSCRIPTION "REP_MSSQL"."SUB_T2_PK" DISTRIBUTE;
-- do some dmls in REP_MSSQL and check change data here
SELECT * FROM "REP_MSSQL"."DPSIMPLE";

-- stop the replication
ALTER REMOTE SUBSCRIPTION "REP_MSSQL"."SUB_T2_PK" RESET;
DROP REMOTE SUBSCRIPTION "REP_MSSQL"."SUB_T2_PK";
DROP TABLE "REP_MSSQL"."V_T2_PK";
DROP TABLE "REP_MSSQL"."T_T2_PK";

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
