# MariaDB Database and User Setup

## Task

Set up a MariaDB database server and configure it for application use.

- Install and configure MariaDB server
- Create a database
- Create a user with a defined password
- Grant full privileges to the user on the database
---

## Steps

### 1. Install MariaDB

```bash
sudo yum install mariadb-server
```

### 2. Enable and Start the Service

```bash
sudo systemctl enable mariadb
sudo systemctl start mariadb
```

### 3. Run the Security Script

```bash
sudo mysql_secure_installation
```

Hardens the installation by setting a root password, removing anonymous users, and disabling remote root login.

### 4. Connect to MariaDB

```bash
sudo mysql
```

### 5. Create the Database

```sql
CREATE DATABASE <database_name>;
```

### 6. Create the User

```sql
CREATE USER '<username>'@'localhost' IDENTIFIED BY <password>;
```

The `@'localhost'` restricts the user to local connections only. MariaDB treats `'user'@'localhost'` and `'user'@'%'` as distinct identities, so the host must always be specified explicitly.

### 7. Grant Full Privileges

```sql
GRANT ALL PRIVILEGES ON <database>.* TO '<username>'@'localhost';
```

The dot notation `database.*` means: all tables within the specified database.

### 8. Verify the Setup

```sql
SHOW GRANTS FOR '<username>'@'localhost';
```

Expected output:

```
GRANT USAGE ON *.* TO '<username>'@'localhost' IDENTIFIED BY PASSWORD '***'
GRANT ALL PRIVILEGES ON '<database>'.* TO '<username>'@'localhost'
```

- `GRANT USAGE ON *.*` – the user exists in the system
- `GRANT ALL PRIVILEGES ON <database>.*` – full access on the target database
---

## Key Concepts

**Host binding in user creation**: The `@'localhost'` part defines where the client connects *from*, not where the server is. Using `@'%'` would allow connections from any host.

**Dot notation in GRANT** — `database.table` syntax targets a specific scope. `<database>.*` means all tables in that database; `*.*` would mean all tables in all databases.

**Multiline prompt** — In the MariaDB shell, `->` instead of `>` means the statement is not yet complete. Every SQL command must end with `;`. Use `\c` to cancel a pending statement.