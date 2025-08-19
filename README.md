# Fixing MariaDB Server Startup Issues

I see you're trying to run a portable MariaDB server but facing issues with both port conflicts and missing database files. Let me help you resolve these problems.

## 1. First, let's create a batch file to manage your server

Create a new file called `start_mariadb.bat` with the following content:

```batch
@echo off
title MariaDB Portable Server Manager
echo ====================================
echo    MariaDB Portable Server Control
echo ====================================
echo.

REM Check if port 3307 is already in use
netstat -ano | findstr :3307 > nul
if %errorlevel% equ 0 (
    echo Port 3307 is already in use!
    echo Finding process using port 3307...
    for /f "tokens=5" %%a in ('netstat -ano ^| findstr :3307') do (
        set pid=%%a
        tasklist /fi "pid eq !pid!" | findstr /i "mysql mariadb"
        if !errorlevel! equ 0 (
            echo Stopping existing MariaDB/MySQL process...
            taskkill /f /pid !pid!
            timeout /t 2 /nobreak > nul
        ) else (
            echo Port 3307 is used by another application (PID: !pid!)
            echo Please free the port and try again.
            pause
            exit /b 1
        )
    )
)

REM Create necessary directories if they don't exist
if not exist "C:\Users\User\Downloads\Html Exe\Html Exe\mariadb\data" (
    mkdir "C:\Users\User\Downloads\Html Exe\Html Exe\mariadb\data"
)
if not exist "C:\Users\User\Downloads\Html Exe\Html Exe\mariadb\tmp" (
    mkdir "C:\Users\User\Downloads\Html Exe\Html Exe\mariadb\tmp"
)

REM Initialize database if not exists
if not exist "C:\Users\User\Downloads\Html Exe\Html Exe\mariadb\data\ibdata1" (
    echo Initializing database...
    "C:\Users\User\Downloads\Html Exe\Html Exe\mariadb\bin\mysql_install_db.exe" --datadir="C:\Users\User\Downloads\Html Exe\Html Exe\mariadb\data" --basedir="C:\Users\User\Downloads\Html Exe\Html Exe\mariadb"
)

REM Start MariaDB server
echo Starting MariaDB server on port 3307...
"C:\Users\User\Downloads\Html Exe\Html Exe\mariadb\bin\mysqld.exe" --defaults-file="C:\Users\User\Downloads\Html Exe\Html Exe\mariadb\my.ini" --console

pause
```

## 2. Create a stop server batch file

Create another file called `stop_mariadb.bat`:

```batch
@echo off
echo Stopping MariaDB server...
taskkill /f /im mysqld.exe
timeout /t 3 /nobreak > nul
echo Server stopped.
pause
```

## 3. Fix your my.ini configuration

Your configuration looks mostly correct, but let's make a few adjustments. Replace your my.ini with this optimized version:

```ini
# =========================
# MariaDB Portable Config
# =========================

[client]
port=3307
socket=mariadb.sock

[mysqld]
# Port default
port=3307
socket=mariadb.sock

# Lokasi instalasi MariaDB portable
basedir="C:/Users/User/Downloads/Html Exe/Html Exe/mariadb"
datadir="C:/Users/User/Downloads/Html Exe/Html Exe/mariadb/data"
tmpdir="C:/Users/User/Downloads/Html Exe/Html Exe/mariadb/tmp"
pid-file="mysql.pid"

# Buffer dan batas dasar (ringan untuk laptop/portable)
key_buffer_size=16M
max_allowed_packet=16M
sort_buffer_size=512K
net_buffer_length=8K
read_buffer_size=256K
read_rnd_buffer_size=512K
myisam_sort_buffer_size=8M

# Log error
log-error="mysql_error.log"

# Plugin directory
plugin-dir="C:/Users/User/Downloads/Html Exe/Html Exe/mariadb/lib/plugin/"

# ID unik server (wajib ada, default 1)
server-id=1

# InnoDB settings
innodb_data_home_dir="C:/Users/User/Downloads/Html Exe/Html Exe/mariadb/data"
innodb_data_file_path=ibdata1:10M:autoextend
innodb_log_group_home_dir="C:/Users/User/Downloads/Html Exe/Html Exe/mariadb/data"
innodb_buffer_pool_size=64M
innodb_log_file_size=16M
innodb_log_buffer_size=8M
innodb_flush_log_at_trx_commit=1
innodb_lock_wait_timeout=50

# Skip problematic tablespace errors
innodb_force_recovery=0

# SQL mode (bisa disesuaikan)
sql_mode=NO_ZERO_IN_DATE,NO_ZERO_DATE,NO_ENGINE_SUBSTITUTION
log_bin_trust_function_creators=1

# Skip name resolution for faster connections
skip-name-resolve

# Allow multiple connections from same IP
bind-address=0.0.0.0

[mysqldump]
quick
max_allowed_packet=16M

[mysql]
no-auto-rehash

[isamchk]
key_buffer=20M
sort_buffer_size=20M
read_buffer=2M
write_buffer=2M

[myisamchk]
key_buffer=20M
sort_buffer_size=20M
read_buffer=2M
write_buffer=2M

[mysqlhotcopy]
interactive-timeout
```

## 4. Database recovery script

If you need to recover the login database, create a file called `recover_login_db.sql`:

```sql
-- First, let's check if the database exists and its current state
SELECT SCHEMA_NAME FROM INFORMATION_SCHEMA.SCHEMATA WHERE SCHEMA_NAME = 'login';

-- If the database exists but tables are corrupted, we'll recreate them
DROP DATABASE IF EXISTS login;
CREATE DATABASE login CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
USE login;

-- Recreate the users table
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    email_verified_at TIMESTAMP NULL,
    password VARCHAR(255) NOT NULL,
    remember_token VARCHAR(100) NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- Recreate the password_resets table
CREATE TABLE password_resets (
    email VARCHAR(255) NOT NULL,
    token VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NULL,
    INDEX (email)
) ENGINE=InnoDB;

-- Recreate the contacts table (if needed)
CREATE TABLE contacts (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    phone VARCHAR(50) NULL,
    message TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- Recreate the password_reset_tokens table (if needed)
CREATE TABLE password_reset_tokens (
    email VARCHAR(255) NOT NULL PRIMARY KEY,
    token VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NULL
) ENGINE=InnoDB;

-- Show the created tables
SHOW TABLES;
```

## 5. How to use these files:

1. Save all the files in your MariaDB directory: `C:\Users\User\Downloads\Html Exe\Html Exe\mariadb\`
2. Double-click on `start_mariadb.bat` to start the server
3. If you need to recover the database:
   - Run the server
   - Connect using: `mysql -u root -P 3307`
   - Run the SQL commands from `recover_login_db.sql`

## 6. Additional troubleshooting:

If you still encounter issues, try these steps:

1. Completely remove the data directory and reinitialize:
   ```batch
   rmdir /s /q "C:\Users\User\Downloads\Html Exe\Html Exe\mariadb\data"
   mkdir "C:\Users\User\Downloads\Html Exe\Html Exe\mariadb\data"
   "C:\Users\User\Downloads\Html Exe\Html Exe\mariadb\bin\mysql_install_db.exe" --datadir="C:\Users\User\Downloads\Html Exe\Html Exe\mariadb\data"
   ```

2. Check if your antivirus is blocking MariaDB files

3. Make sure you have the correct version of Visual C++ Redistributable installed

The batch files will automatically handle port conflicts and directory creation, making your portable MariaDB server much easier to manage.
