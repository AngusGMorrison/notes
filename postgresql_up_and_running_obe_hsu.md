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