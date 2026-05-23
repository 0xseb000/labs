# PostgreSQL Database and User Setup

## Task

Set up a PostgreSQL database server as a pre-requisite for an application deployment. The PostgreSQL service was already installed on the database server.

- Create a database user with a defined password
- Create a database and grant full privileges to user
---

## Steps

### 1. Identify the OS

```bash
cat /etc/os-release
```

> note to myself: `uname -v` shows the kernel build string (compiler, build host, timestamp). Not the installed distribution. `/etc/os-release` is the reliable source.

---

### 2. Create the Database User

PostgreSQL provides a `createuser` shell utility. Since only the `postgres` system user has direct database access by default, `sudo -u` is used to run the command as that user.

```bash
sudo -u postgres createuser --interactive
# Enter name of role to add: <username>
# Shall the new role be a superuser? (y/n): y
```

`--interactive` prompts for the username and role type via CLI. No SQL required. However, it does not handle password setting. That requires a separate step. 

---

### 3. Set the User Password

Connect to PostgreSQL as the `postgres` user:

```bash
psql postgres postgres
```

Then set the password using the psql meta-command:

```sql
\password <username>
```

`\password` prompts for the new password and stores it securely as a hash. No plaintext in the shell history. 

---

### 4. Create the Database

Exit psql and create the database from the shell:

```bash
sudo -u postgres createdb <db-name>
```

---

### 5. Grant Full Privileges

Re-enter psql and grant all privileges on the database to the user:

```bash
psql postgres postgres
```

```sql
GRANT ALL PRIVILEGES ON DATABASE <db-name> TO <username>;
``` 

---

### 6. Verify

```sql
\l <db-name>
```

Expected output includes `<username>=CTc/postgres` in the Access privileges column, confirming the grant was applied.

> `C` = CONNECT, `T` = TEMPORARY, `c` = CREATE. These are the standard database-level privileges granted by `ALL PRIVILEGES`. 

---

## Key Concepts

| Concept                            | Notes                                                                                |
|------------------------------------|--------------------------------------------------------------------------------------|
| `postgres` system user             | Default Linux user with passwordless PostgreSQL admin access                         |
| `createuser`                       | Shell utility to create a PostgreSQL role without writing SQL                        |
| `createdb`                         | Shell utility to create a PostgreSQL database                                        |
| `\password`                        | psql meta-command to set a user password securely                                    |
| `GRANT ALL PRIVILEGES ON DATABASE` | Grants CONNECT, TEMPORARY, and CREATE on a database                                  |
| `\l`                               | Lists databases with owner, encoding, and access privileges                          |
| `\du`                              | Lists all users                                                                      |
| `uname -v` vs `/etc/os-release`    | Kernel build string vs actual OS. Always use os-release to identify the distribution |