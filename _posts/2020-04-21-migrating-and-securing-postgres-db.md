---
title:  "Migrating and securing a PostgreSQL database"
date: 2020-04-21 16:10:00 -0000
categories: Database PostgreSQL
---

# Migrating and securing a PostgreSQL database

## Scenario
I need to update the PostgreSQL version of a Azure Database for PostgreSQL server.

 * the server is currently running v9 and needs to go to v11.
 * Azure Database for PostgreSQL servers cannot be upgraded in place.
 * the server was incorrectly configured: the sole database on the server is under the `public` schema in the default `postgres` database.
 * the permissions model was also configured incorrectly: the services accessing this database rely on a user with full schema permissions - this represents a security threat.

## Objectives:
 * Create new Azure Database for PostgreSQL server at version 11.
 * Create a new database and schema for data to be migrated.
 * Setup an appropriate permission model to minimise user privilege:
    * an admin user
    * a read/write user for CRUD operations only
    * a user for performing migrations
 * Migrate the data from the old server to the new server.

## Procedure:
### Creating a new server and database
This could be done programatically but for this exercise, it can be easily achieved through the portal.
 1. Create a new Azure Database for PostgreSQL server with the following properties:
    * Type: Single server
    * Subscription: `<subscription>` (as per environment)
    * Resource group: `<resource group>` (as per environment)
    * Server name: `<server name>` e.g. `psql-rafb-dev` (as per naming conventions)
    * Data source : `None`
    * Location: `(Europe) UK South`
    * Version: `11`
    * Compute + storage: 
      * Type: `Basic`  
      * Storage: `50 GB`
      * Backup Retention Period: `14 Days`
    * Administrator account:
      * Admin username `<username>` (Store in KeePass)
      * Password: `<password>` (Store in KeePass)
 1. Once deployed, update `Connection security` to whitelist the IP address of the machine from which to perform the migration.

### Creating the permissions model
New databases are created with a default `public` schema. When using PG Admin, it is easier to restore to a schema with same name and then rename the schema afterwards. The steps below, therefore, setup the user permissions model on the public schema:
 1. Connect to the new server with PG Admin, using the administrator account details entered above.
 1. Create a new database, e.g. `rafb`.
 1. Run the following query on the public schema of the new database:
    ```TSQL
    CREATE ROLE read_write;

    GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO read_write;  
    GRANT SELECT, USAGE ON ALL SEQUENCES IN SCHEMA public TO read_write;

    ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO read_write;  
    ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, UPDATE ON SEQUENCES TO read_write;

    CREATE USER read_write_user WITH PASSWORD '<password1>';  
    GRANT read_write TO read_write_user;

    CREATE ROLE read_write_migrate;

    GRANT ALL ON SCHEMA public TO read_write_migrate;  
    GRANT read_write to read_write_migrate;

    CREATE USER read_write_migrate_user WITH PASSWORD '<password2>';  
    GRANT read_write_migrate TO read_write_migrate_user;
    ```
### Migrating the database
This could be done programatically 
 1. Run the following command from a command line, substituting hostname, passwords etc. as appropriate:  
 ```Shell
 pg_dump -d "user=postgresadmin@dev-temp-store password=<password1>  host=dev-temp-store.postgres.database.azure.com port=5432 dbname=postgres sslmode=require" | psql -d "user=postgres@psql-rafb-dev password=<password2> host=psql-rafb-dev.postgres.database.azure.com port=5432 dbname=rafb sslmode=require"
 ```

Alternatively it can be be easily achieved through PG Admin: 
 1. Connect to the **old** server with PG Admin, using appropriate administrator account details.
 1. Right-click on the `public` schema and click `Backup`.
 1. Select the following options:
    * Filename: `<filename>.backup`
    * Format: `Custom`
    * Encoding: `UTF8`
    * Sections
      * Pre-data
      * Data
      * Post-data
    * Don't save
      * Owner
 1. Click `Backup` and verify process completes successfully.
 1. Connect to the new server with PG Admin, using the administrator account details entered previously.   
 1. Right-click on the `public` schema and select `Restore`.
 1. Select the following options:
    * Filename: `<filename>.backup` (select file created earlier)
    * Sections
      * Pre-data
      * Data
      * Post-data
    * Don't save
      * Owner
    * Queries
      * Clean before restore  

Either way, rename the schema afterwards as follows:
 1. Right-click on the `public` schema and select `Properties`.
 1. Amend the name as appropriate and click `OK`.

 ## Test
 Ensure the changes have been effective with PG Admin, or from the command prompt with pgcli:
 1. From a command prompt, run the following command, substituting in the appropriate host and password etc.:  
```Shell
pgcli postgres://postgresadmin@dev-temp-store:<password>@dev-temp-store.postgres.database.azure.com:5432/postgres?sslmode=require
```
1. Then the following:  
```TSQL
SELECT * FROM public.registrations LIMIT 10
```
