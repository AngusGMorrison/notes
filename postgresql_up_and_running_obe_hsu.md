# PostgreSQL: Up and Running
*Regina Obe & Leo Hsu*

# Chapter 1: The Basics

## PostgreSQL Database Objects
* Databases: Each PostgreSQL service houses many individual databases.
* Schemas:
  * If the database is a country, schemas are the states.
  * Most database objects belong first to a schema, then to a database.
  * When you create a new database, Postgres automatically creates the `public` schema to store objects you create.
    * If you have many tables, they should be organized into different schemas.
* Tables:
  * The first citizens of their respective schemas.
  * Inheritable: streamlines database time and saves looping code when querying tables with nearly identical structures.
  * Postgres creates a matching data type whenever a table is created.
* Views:
  * Used to present queries from multiple tables or with additional derived columns based on complex calculations.
  * Postgres allows updating of underlying data via the view if the view draws from a single table.
    * To update data from views that join multiple tables, create a trigger against the view.
    * Materialized views cache data to speed up commonly used queries at the sacrifice of not having the latest data.
* Extensions:
  * Allow developers to package functions, data types, casts, custom index types, etc. for installation and removal as a unit.
  * Once installed, extensions must be enabled for each database separately.
  * When enabling an extension, you choose the schema in which its objects reside.
    * The default is the `public` schema, which clutters it.
    * Create a separate schema to hold extensions.
    * If the extension is particularly large, create a schema specifically for it.
    * Append the extension schema to the database's `search_path` variable to refer to it without prepending the schema name.
    * Some extensions dictate which schema they are stored in, especially procedural languages (PL).
  * `CASCADE` ensures extension dependencies are automatically installed:
    `CREATE EXTENSION postgis_tiger_geocoder CASCADE;`
* Functions: Program custom data manipulation routines.
* Languages:
  * Create functions using a procedural language (PL).
  * SQL, PL/pgSQL, and C are installed by default.
  * Install additional languages using the extension framework or the `CREATE PROCEDURAL LANGUAGE` command.
* Operators:
  * Symbolically named aliases for functions.
  * If creating custom data types, you'll want to create your own +, -, = operators, etc.
* Foreign tables and foreign data wrappers:
  * Foreign tables are virtual tables linked to data outside Postgres, e.g. CSVs, a Postgres table on another server, a table in a different DBMS, or a web service.
  * Foreign data wrappers facilitate the handshake between Postgres and external services.
    * Follow that SQL/Management of External Data (MED) standard.
* Triggers and trigger functions:
  * Triggers detect data change events.
  * Postgres fires a trigger, and you can execute a trigger function in response.
  * Triggers can run in response to particular types of statements, changes to particular rows or columns, and can fire before or after a data-change event.
  * Trigger functions have access to special varaibles that store the data both before and after the triggering event.
    * Often using to write complex validation routines beyond what can be accompished with check constraints.
* Catalogs:
  * System schemas that store Postgres built-in functions and metadata.
  * `pg_catalog`: holds all functions, tables, system views, casts and types packaged with Postgres.
  * `information_schema`: offers views exposing metadata in a format dictated by the ANSI SQL standard.
* Types:
  * Data types, including composite types such as complex numbers or vectors.
  * When you create a table, a composite type based on the structure of the table is automatically created.
    * Useful when looping through tables.
* Full text search (FTS):
  * Natural language-based search which matches based on semantics.
  * Faciliated by FTS configurations, FTS dictionaries and FTS parsers.
  * FTS is built-in, but for specialized vocabulary, the packages objects must be swapped out.
* Casts: prescribe how to convert from one data type to another.
* Sequences:
    * Control autoincrementation of serial data types.
    * More than one table can share the same sequence object.
* Rules:
  * Instructions to rewrite a query prior to execution.
  * Made redundant by triggers.
* The term `GUC` stands for grand unified configuration: the configuration settings in Postgres.


# Chapter 2: Database Administration

## Configuration Files
* postgresql.conf:
  * Controls general settings, such as memory allocation, default storage location for new databses, the IP addresses that Postgres listens on, locations of logs, and more.
* pg_hba.conf:
  * Controls access to the server, dictating which users can log in to which databases, which IP addresses can connect, and which authentication scheme to accept.
* pg_ident.conf:
  * Maps an authenticated OS login to a Postgres user.
  * "User" refers to a role with login privileges.
* To find the location of these files:
  `SELECT name, setting FROM pg_settings WHERE category = 'File Locations';`

### Making Confgurations Take Effect
* If ensure whether a change requires a reload or restart, check the context setting associated with the configuration.
  * `postmaster`: restart.
  * `user`: reload.
* Reloading:
  * New users connecting after a reload will receive the new setting.
  * Several approaches:
    * `pg_ctl reload -D your_data_directory`

    * With pgAdmin.
* Restarting:
  * Closes any active connections.
  * `pg_ctl restart -D your_data_directory`

### The postgresql.conf File
* Don't edit directly: override settings in `postgresql.auto.conf`.
* View settings without opening config files by querying the `pg_settings` view:
  ```SQL
  SELECT
    name,
    context,
    unit,
    setting, boot_val, reset_val
  FROM pg_settings
  WHERE name IN ('listen_addresses', 'deadlock_timeout', 'shared_buffers', 'effective_cache_size', 'work_mem', 'maintenance_work_mem')
  ORDER BY context, name;
  ```
  * `context`: the scope of the setting.
    * `user`: affects just the changing user's session.
    * `superuser`: applies to all users who connect after a reload.
    * `postmaster`: the Postres service; affects the entire server.
    * Settings set at database, user, session and function levels don't require a reload.
    * Settings set at the database level take effect on the next connection.
    * Settings for the session or function take effect immediately.
  * `unit`:
    * To see the total value of a setting that has a unit of 8kB, use `SHOW setting_name;` or `SHOW ALL;`.
  * `setting`: The current setting.
  * `boot_val`: The default setting.
  * `reset_val`: The new setting if you restart or reload.
* The `pg_file_settings` system view is used to query settings, including the source file and line.
  * `applied`: whether the setting is in effect.
  * Where a setting is present in both `postgresql.conf` and `postgresql.auto.conf`, the latter takes precedence.
  *
    ```SQL
    SELECT name, sourcefile, sourceline, setting, applied
    FROM pg_file_settings
    WHERE name IN ('listen_addresses',...)
    ORDER BY name;
* The following settings will prevent clients from connecting if set wrongly:
  * All require a restart.
  * `listen_addresses`:
    * Which IPs to listen on.
    * Default to `local` or `localhost`.
    * `*` means all available IP addresses.
  * `port`: Defaults to 5432.
  * `max_connections`: max concurrent connections allowed.
  * `log_destination`:
    * Specfies the format of logfiles rather than their physical location.
    * Defaults to `stderr`.
* The following settings affect performance:
  * Default settings are rarely optimal.
  * `shared_buffers`:
    * Amount of memory shared among all connections to store recently accessed pages.
    * Profoundly affects speed of queries
    * Set around 25% of available RAM.
    * Diminishing returns > 8 GB.
    * Changes require restart.
  * `effective_cache_size`:
    * An estimate of how much memory Postgres expects the OS to devote to it.
    * Has no effect on actual allocation.
    * Used by query planner to guess whether intermediate steps and query ouput will fit in RAM.
    * If too low, indexes may be underused.
    * Setting to half the RAM of a dedicated server is a good starting point.
    * Changes require a reload.
  * `work_mem`:
    * Max memory allocated for each operation, e.g. sorting, hash join, table scan.
    * Optimal setting depends on how the database is used.
      * For many users with simple queries, set low to be democratic, or first user will hog memory.
      * Understanding work mem: http://bit.ly/15SWsHh.
    * Change requires reload.
  * `maintenance_work_mem`:
    * Total memory for housekeeping, e.g. vacuuming.
    * Don't set higher than 1 GB.
    * Change requires reload.
  * `max_parallel_workers_per_gather`:
    * Determines the max parallel worker threads that can be spawned for each gather operation.
    * Default is 0 – no parallelism.
    * Increase for machines with more than one core.
    * Must be less than `max_worker_processes`, which is a superset.
      * Defaults to 8.
      * `max_parallel_workers` controls the subset of `max_worker_processes` allocated for parallelization.
* Changing postgresql.conf settings:
  * `ALTER SYSTEM SET work_mem = '500MB';`
  * Changes are made to `postgresql.auto.conf`.
  * Consider organizing settings into multiple configuration files and linking them back to `postgresql.conf` using `include 'filename'` or `include_if_exists`.
    * Filename can be an absolute path or relative from the `postgresql.conf` file.
* If the server won't start after edits:
  * Look at the logfile in the root of the data folder or in the `pg_log` subfolder.
  * Shared buffers may be set too high.
  * An old `postmaster.pid` may be leftover from a failed shutdown. Delete this file.
    * Located in the data cluster folder.

### The pg_hba.conf File
* Controls which IP addresses and users can connect to the database.
* Dictates the authentication protocol that the client must follow.
* Changes require at least a reload.
* `METHOD`: authentication method, typically ident, trust, md5, peer and password.
* `ADDRESS`: IPv4 or IPv6 syntax for defining network ranges.
* `TYPE`:
  * `host` or `hostssl`.
  * SSL settings can be found in `postgres.conf`: `ssl`, `ssl_cert_file`, `ssl_key_file`.
* `DATABASE`: when set to `replication`, specifies the range of IP addresses allowed to replicate with this server.
* For each connection, `pg_hba.conf` is checked from the top down.
  * Server reads no further once it encounters a rule that either allows or disallows a connection.
* The system view `pg_hba_file_rules` lists the contents to the `pg_hba.conf` file.
* If the server won't start after editing `pg_hba.conf`:
  * Usually caused by typos or adding an unavailable authentication scheme.
  * When `pg_hba.conf` can't be parsed, Postgres blocks all access for safety.
  * Read the logfile in the data folder or `pg_log` subfolder.
* Authentication methods:
  * Users or devices must still satisfy role and database access restrictions after being admitted by `pg_hba.conf`.
  * `trust`:
    * Least secure; no password required.
    * Only for local or private network connections.
    * Discouraged in production.
  * `md5`:
    * Very common.
    * Requires an md5-encrypted password to connect.
  * `password`: Clear-text password authentication.
  * `ident`:
    * Uses `pg_ident.conf` to check whether the OS account of the user trying to connect has a mapping to a Postgres account.
    * Password is not checked.
  * `peer`: Uses the OS name of the user from the kernel.
  * `cert`:
    * Connections must use SSL.
    * Client must have a registered certificate.
    * Uses an ident file such as `pg_ident` to map the certificate to a PostgreSQL user.

## Managing Connections
* It is appropriate to cancel all active update queries before backing up the database and before restoring the database.
* To cancel running queries and terminate connections:
  1. Retrieve a listing of recent connections and PIDs:
    `SELECT * FROM pg_stat_activity`
    * `pg_stat_activity` is a view that lists the last query running on each connection, the connected user, the databse and the start times of the queries.
    * Identifiy the PIDs to terminate.
  2. Cancel active queries for each PID:
    `SELECT pg_cancel_backend(1234);`
  3. Terminate the connection:
    `SELECT pg_terminate_backend(1234);`
      * You may need to additionally terminate the client connection to prevent it from reconnecting:
        `REVOKE CONNECT ON DATABASE dbname FROM PUBLIC, username;`
* To kill all connections other than your own:
  ```SQL
  SELECT pg_terminate_backend(pid)
  FROM pg_stat_activity
  WHERE pid <> pg_backend_pid()
  AND datname = 'database_name';
  ```
* Certain operational parameters can be set at the server, database, user, session and function level.
  * Queries exceeding the parameters will be cancelled.
  * `deadlock_timeout`:
    * The amount of time a deadlocked query should wait before giving up.
    * Defaults to 1000 ms.
    * If your app performs many updates, consider increasing to minimize contention.
    * `SELECT FOR UPDATE NOWAIT` automatically cancels the query on deadlocking.
    * `SELECT FOR UPDATE SKIP LOCKED` skups over locked rows.
  * `statement_timeout`:
    * The amount of time a query can run before being cancelled.
    * Defaults to 0 (no limit).
    * Cancelling a function cancels the query and the transaction that's calling it.
  * `lock_timeout`:
    * The amount of time a query should wait for a lock before giving up.
    * Most applicable to update queries, which require an exclusive lock.
    * Defaults to 0 (wait infinitely).
    * Should be lower than `statement_timeout`, or `lock_timeout` will never occur.
  * `idle_in_transaction_session_timeout`:
    * The amount of a time a transaction can stay idle before it is terminated.
    * Defaults to 0 (wait infinitely).
    * Useful for preventing queries from holding onto locks indefinitely.
  
### Check for Queries Being Blocked
* `wait_event_type` and `wait_event` in the `pg_stat_activity` view provide information about what resource a query was waiting for.

## Roles
* Roles that can login are called login roles.
* Roles that contain other roles are called group roles.
* Group roles that can login are group login roles.

### Creating Login Roles
* When you initialize the data cluster during setup, Postgres creates a single login role with the name `postgres`.
  * Also creates a namesake database called `postgres`.
* You can bypass the password setting by mapping an OS root user to the new role and using `ident`, `peer` or `trust` for authentication.
* After installing Postgres, log in as postgres and create other roles.
* 
  ```SQL
  CREATE ROLE leo LOGIN PASSWORD 'king' VALID UNTIL 'infinity' CREATEDB;
  ```
  * `VALID UNTIL` is optional; defaults to infinity.
  * `CREATEDB` grants database creation priviliges to the new role.
  * To create roles that cannot login, omit `LOGIN PASSWORD`.

### Creating Group Roles
* Group roles serve as containers for other roles.
*
  ```SQL
  CREATE ROLE royalty INHERIT;
  ```
  * `INHERIT`/`NOINHERIT`: Any member of ryalty will automatically inherit the privileges of the royalty role.
* To add members to a group role:
  ```SQL
    GRANT royalty TO leo;
  ```
* Superuser privileges can't be inherited, but `SET ROLE` can be used to impersonate a group role.
  * Impersonating grants to privileges for the duration of the session.
  * Available to all users.
  * Sets the `current_user` global variable.
  *
    ```SQL
    SET ROLE royalty;
    ```
  * Non-superusers can `SET ROLE` only to the role the `session_user` is or the roles the `session_user` belongs to.
  * Superusers can `SET ROLE` to any other role.
  * Grants the privileges of the impersonated user except for `SET SESSION AUTHORIZATION` and `SET ROLE`.
* `SET SESSION AUTHORIZATION` is available to people who log in as superusers.
  * Sets both `current_user` and `session_user` to the values of the user being impersonated.
  * A `session_user` that has superuser rights can `SET ROLE` to any other role.
  * `SET SESSION AUTHORIZATION` remains availale even if you impersonate a user who is not a superuser.
* To give a role more privileges:
  ```SQL
  ALTER ROLE royalty SUPERUSER;
  ```

## Database Creation
* The minimum command to create a DB:
  ```SQL
  CREATE DATABASE mydb;
  ```

### Template Databases
* Serve as skeletons for new databases.
* Postgres copies all settings and data from the template to the new database.
* Postgres comes with two templates by default:
  * `template1`: The default template when creating any new database.
    * You can't change encoding and collation of a database created from `template1`.
  * `template0`:
    * The immaculate model that serves as a backup for the original template1. Do not alter.
    * Create new databases that require encoding and collation changes from `template0`.
* To create a database using a specific template:
  ```SQL
  CREATE DATABASE mydb TEMPLATE my_template_db;
  ```
* When a database is marked as a template, it is no longer editable and deletable.
  * Any role with the `CREATEDB` privilege can use a template database.
  *
    ```SQL
    UPDATE pg_database SET datistemplate = TRUE WHERE datname = 'mydb';
    ```

### Using Schemas
* The `user` global variable is an alias for `current_user`.
  * Retrieves the role currently logged in.
* The default search path is: `search_path = "$user", public`.
  * Each user's schema is checked before the public schema.
  * Enables scripts to work for different users without any changes.
* Before installing extensions:
  * Create a new schema: `CREATE SCHEMA my_extensions;`
  * Add the new schema to the search path: `ALTER DATABASE mydb SET search_path='$user', public, my_extensions;`.
    * Will not affect existing connections.

## Privileges

### Types of Privileges
* Basic:
  ```SQL
  SELECT
  INSERT
  UPDATE
  ALTER
  EXECUTE
  DELETE
  TRUNCATE
  ```
* Most privileges have a context, e.g. `GRANT SELECT ON table`.

### Getting Started
1. Postgres creates one superuser and one database at installation, both named postgres.
2. Before creating your first database, create a role that will own the database and can log in, such as:
  `CREATE ROLE mydb_admin LOGIN PASSWORD 'something';`
3. Create the database and set the owner:
  `CREATE DATABASE mydb WITH owner = mydb_admin;`
4. Log in as the mydb_admin user and start setting up schemas and tables.

### GRANT
* The primary means to assign privileges:
  `GRANT some_privilege TO some_role;`
* You need to have the privilege you're granting.
* You must have the `GRANT` privilege.
* Some privileges, like `DROP` and `ALTER` always remain with the owner of the object and can't be given away.
* The owner of an object has all privileges.
  * Ownership does not drill down to child objects – the owner of a database may not own all the schemas inside it.
* `WITH GRANT OPTION` enables the grantee to grant the privilege to others in turn.
  `GRANT ALL ON ALL TABLES IN SCHEMA public TO mydb_admin WITH GRANT OPTION;`
* Use `ALL` to grant privileges on all objects of a specific type.
* To grant privileges to all roles, use `PUBLIC`:
  `GRANT USAGE ON SCHEMA my_schema TO PUBLIC;`
* By default, `CONNECT`, `CREATE TEMP TABLE` for databases and `EXECUTE` for functions are granted to `PUBLIC`.
* Remove privileges with `REVOKE`:
  `REVOKE EXECUTE ON ALL FUNCTIONS IN SCHEMA my_schema FROM PUBLIC;`

### Default Privileges
* Defaults allow you to set privileges before specific objects' creation.
* 
  ```SQL
  GRANT USAGE ON SCHEMA my_schema TO PUBLIC;

  ALTER DEFAULT PRIVILEGES IN SCHEMA my_schema
  GRANT SELECT, REFERENCES ON TABLES TO PUBLIC;
  ```
* `GRANT USAGE` on a schema is the first step to granting access to objects in the schema.
  * Required to perform any action in the schema.

### Privilege Idiosyncrasies
* Owners of a database do not have access to all objects in the database.

## Extensions
* Use templates to ensure that all databases have a certain set of extensions.
* To view installed extensions:
  ```SQL
  SELECT name, default_version, installed_version, left(comment, 30) AS comment
  FROM pg_available_extensions
  WHERE installed_version IS NOT NULL
  ORDER BY name;
  ```
  * Remove the `IS NOT NULL` clause to see all extensions on the server, regardless of if they are installed in the current database.
* `\dx+ extension_name` reveals more information about an installed extension.

### Installing Extensions
1. Installing on the server:
  * Download the binary files and libraries.
  * Copy the binaries to the `bin` and `lib` folders.
  * Copy to script files to `share/extension`.
2. Installing into a database:
  * `CREATE EXTENSION extension_name SCHEMA my_extensions`;
  * Uninstall with `DROP EXTENSION`.
  * To install using psql: `psql -p 5432 -d mydb -c "CREATE EXTENSION extension_name";`

## Backup and Restore
* Postgres ships with three utilities for backup: pg_dump, pg_dumpall, pg_basebackup.
  * All found in Postgres `bin` folder.
* Consider creating a `~/.pgpass` file (http://bit.ly/12scPrZ) to store all passwords.
  * pg_dump and pg_dumpall don't have password options.
  * Alternatively, set a password in the `PGPASSWORD` env var.

### Selective Backup Using pg_dump
* Selectively backs up tables, schemas and databases.
* Backs up to SQL, compressed, TAR and directory formats.
  * Compressed, TAR and directory backups can be parallel restored by pg_restore.
  * Directory backups allow parallel pg_dump of a large database.
    * Each table is backed up in a separate gzipped file in a folder.
    * Gets around file size limitations.
    * The `--jobs` or `-j` option specifies the number of backups to run in parallel in directory mode.

### System-wide Backup Using pg_dumpall
* Backs up all databases on a server into a single plain-text file.
* Includes server globals such as tablespace definitions and roles.
  * Recommended to back up globals with pg_dumpall daily:
    `pg_dumpall -h localhost -U postgres --port=5432 -f myglobals.sql --globals-only`.
* Not recommended for database backups as restoring from a huge plain-text backup is a pain.
* pg_basebackup in conjunction with streaming replication is the fastest way to recover from major server failure.

### Restoring Data
* Two ways to restore data:
  * Use psql to restore plain-backups generated with pg_dumpall or pg_dump.
    * With SQL backups, you must execute the entire script; no cherry-picking objects.
    * To restore a backup and ignore errors: `psql -U postgres -f myglobals.sql`
    * To restore, stopping if an error is found: `psql -U postgres --set ON_ERROR_STOP=on -f myglobals.sql`
    * To restore to a specific database: `psql -U postgres -d mydb -f select_objects.sql`
  * Use pg_restore to restore compressed, TAR, and directory backups created with pg_dump.
    * Perform parallel restores using the `-j` (`--jobs=`) option to indicate the number of threads to use.
    * Generate a table of contents file from your backup to check what has been backed up.
      * Editable; allows you to control what gets restored.
    * Selectively restore, even from within a backup of a full database.
    * Backwards compatible.
    * To perform a restore with pg_restore:
      1. Create the databse anew using SQL.
      2. Restore with `pg_restore --dbname=mydb --jobs=4 --verbose mydb.backup`.
    * If the name of the database is the same as the one you backed up, you can create and restore in one step:
      `pg_restore --dbname=postgres --create --jobs=4 --verbose mydb.backup`
    * A restore will not usually recreate objects already present in the database, but it can using the `--clean` switch.
    * Use `--section=pre-data` to restore the structure of the database without the data.

## Managing Disk Storage with Tablespaces
* Tablespaces ascribe logical names to physical locations on disk.
* Two tablespaces by default:
  * pg_default: Stores all user data.
  * pg_global: Stores system data.
  * Both located in same folder as default data cluster.

### Creating Tablespaces
* Specify a logical name, a folder, and ensure the postgres service account has full access to the physical folder.
* `CREATE TABLESPACE secondary LOCATION '/usr/data/...';`

### Moving Objects Among Tablespaces
* To move all objects in the database to your secondary tablespace:
  `ALTER DATABASE mydb SET TABLESPACE secondary;`
* To move just one table:
  `ALTER TABLE mytable SET TABLESPACE secondary;`
* To move objects from one tablespace to another:
  `ALTER TABLESPACE pg_default MOVE ALL TO secondary;`
  * Must be superuser to move all objects, otherwise only owned objects will be moved.

## Verboten Practices

### Don't Delete Postgres Core System Files and Binaries
* The pg_log folder, found in the data folder, can be purged without harm.
* pg_wal (pg_xlog in older versions) stores transaction logs and should not be deleted.
* pg_xact (pg_clog in older versions) is the active commit log and must never be deleted.

### Don't Grant Full OS Administrative Privileges to the Postgres System Account (postgres)
* Granting unnecessary access leaves your system vulnerable if you fall victim to an SQL injection attack.

### Don't Set shared_buffers as high as Physical RAM
* Server may refuse to start.

### Don't Try to Start Postgres on a Port Already in Use
* If the port is in use, the error in pg_log will be: `make sure PostgreSQL is not already running`.
* Happens if:
  * You've already started the Postgres service.
  * You are trying to run Postgres on a port in use by another service.
  * The postgres service had a sudden shutdown and there is an orphan `postgresql.pid` file in the data folder.
    * Delete the file and try again.
  * There is an orphaned Postgres process.
    * Kill all running Postgres processes and try again.

# Chapter 3: psql

## Environment Variables
* Avoid specifying connection settings for psql by setting `PGHOST`, `PGPORT` and `PGUSER` env vars.
* For more secure access, create a password file (http://bit.ly/12scPrZ).
* `PSQL_HISTORY`: Sets the name of the the psql history file.
* `PSQLRC`: Specifies the location and name of a custom configuration file.
  * psql reads settings from here at startup.

## Interative vs Noninteractive psql
* `\?` brings up a list of available commands.
* `\h` followed by a command shows the Postgres documentation for the command.
* To run commands repeatedly or in sequence, create a script and run it with psql noninteratively.
  * `psql -f <path_to_script_file>`
  * `psql -d my_db -c "DROP TABLE IF EXISTS angus; CREATE SCHEMA staging;"`

## psql Customizations
* Settings changed within a session will only last for the duration of a session.
* Settings specified in `psqlrc` will be loaded each time psql loads.
* To reset a config variable, use `\unset <setting>`.
* When using `\set`, use all caps to set system options and lowercase for your own variables.

### Timing Executions
* `\timing` toggles query timing on and off.

### Autocommit Commands
* Autocommit is on by default.
  * Each command is in its own transaction and is irreversible.
* `\set AUTOCOMMIT off` disables autocommit, allowing you to manually `ROLLBACK;` or `COMMIT;` large batch transactions.

### Shortcuts
* Store useful keyboard shortcuts in `psqlrc`.
  * E.g. `\set eav 'EXPLAIN ANALYZE VERBOSE`.
* Use a colon to resolve the variable:
  * `:eav SELECT COUNT(*) FROM pg_tables;`

### Retrieving Prior Commands
* `HISTSIZE` determines the number of previous commands that can be recalled.
* To pipe the command history into a file for later reference:
  `\set HISTFILE ~/.psql_history - :DBNAME`.

## psql Gems

### Executing Shell Commands
* Run shell commands with `\!`:
  `\! dir`

### Watching Statements
* `watch` repeatedly runs an SQL statement at fixed intervals so you can monitor output.
  *
    ```SQL
    SELECT datname, query
    FROM pg_stat_activity
    WHERE state = 'active' AND pid != pg_backend_pid();
    \watch 10
    ```
* `watch` can also be used for performing an action at regular intervals:
  *
    ```SQL
    SELECT * INTO log_activity
    FROM pg_stat_activity;

    INSERT INTO log_activity
    SELECT * FROM pg_stat_activity; \watch 5
    ```
    * Only the second statement is repeated.
* Kill `watch` with `CTRL-X CTRL-C`.

### Retrieving Details of Database Objects
* `\d` is used to describe various objects.
* `\d+` provides additional details.

### Crosstabs
* Cross tabulation outputs a table where the first column serves as a row header, the second column as a column header, and the third as the value that goes in each cell.
*
  ```SQL
  SELECT student, subject, AVG(score)::numeric(5,2) AS avg_score
  FROM test_scores
  GROUP BY student, subject
  ORDER BY student, subject
  \crosstabview student subject avg_score
  ```

### Dynamic SQL Execution
* You can execute generated SQL in a single step with the `\gexec` command, which iterates through each cell of your query and executes the SQL inside.
  * Iteration is by row, then column.
  * `gexec` is oblivious to the result of the SQL execution.
* This example creates two tables in inserts one row into each table:
  *
    ```SQL
    SELECT
      'CREATE TABLE ' || person.name || ' (a integer, b integer)' AS create,
      'INSERT INTO ' || person.name || ' VALUES(1,2) ' AS insert
    FROM (VALUES ('leo'),('regina')) AS person (name) \gexec
    ```

## Importing and Exporting Data
* `\copy` lets you import data from and export data to a text file.
  * Tab is the default delimiter.
  * Newlines must separate the rows.

## psql Import
* Before loading denormalized or unfamiliar data, create a staging schema to accept it, then write explorative queries.
  * Distribute the data into normalized production tables and delete the staging schema.
* psql processes the entire import as a single transaction.
  * An error causes the whole import to fail.
*
  ```SQL
  \connec† postgresql_book
  \cd /postgresql_book/ch03
  \copy staging.factfinder_import FROM DEC_10_SF1_QTH_with_ann.csv CSV
  ```
* If the file has nonstandard delimiters:
  `\copy sometable FROM somefile.txt DELIMITER '|';`
* To replace `NULL` values during import:
  `\copy sometable FROM somefile.txt NULL AS '';
* `\copy` is not the same as the Postgres `COPY` command.
  * `\copy` interprets all paths relative to the connected client.
  * `COPY` must be able to find its input file in a path accessible by the postgres service account.

### psql Export
*
  ```SQL
  \connect postgresql_book
  \copy (SELECT * FROM staging.factfinder_import WHERE s01 ~ E'^[0-9]+')
  TO '/test.tab'
  WITH DELIMITER E'\t'
  ```
  * Tab-delimited format does not export header columns; must use CSV for headers:
    `WIH CSV HEADER`.

### Copying from or to Program
* psql can fetch data from the output of command-line programs such as `curl` and `ls`, and dump the data into a table.
*
  ```SQL
  \connect postgresql_book
  CREATE TABLE dir_list (filename text);
  \copy dir_list FROM PROGRAM 'dir C:\projects /b'
  ```

## Basic Reporting
* psql can create HTML reports:
*
  ```bash
  psql -d postgresql_book -H -c "
  SELECT category, COUNT(*) AS num_per_cat
  FROM pg_settings
  WHERE category LIKE '%QUERY%'
  GROUP BY category
  ORDER BY category;
  " -o test.html