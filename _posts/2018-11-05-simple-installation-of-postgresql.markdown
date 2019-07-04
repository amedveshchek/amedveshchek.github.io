---
title: "Default PostgreSQL setup on `localhost` (Ubuntu 16.04)"
layout: post
date: 2017-01-05 21:05
image: /assets/images/postgresql-logo.png
headerImage: true
tag:
- postgres
- setup
category: blog
author: alexmedveshchek
description: Default PostgreSQL setup on `localhost` (Ubuntu 16.04)
---

* Install PostgreSQL:
```bash
$> sudo apt-get install postgresql
```

* Login as a default user and show version just for control:
```sql
$> sudo -u postgresql psql
postgres=# select version();
                                                  version                                                      
-------------------------------------------------------------------------------------------------------------------
PostgreSQL 9.5.17 on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 5.4.0-6ubuntu1~16.04.10) 5.4.0 20160609, 64-bit
(1 row)
```
:pushpin: Note that:
  - default authentication for Postgre is the _**ident**_ method, which means authenticate using system user which is mapped to the same user in DB;
  - default user for Postgre is `postgresql`, which is created by installer in DB as well as in your system;
  - the linux user `postgresql` is locked for security reasons, so it doesn't have password; you can only do `su` or `sudo`.
<br/>
<br/>

* Create non-default `testuser`:
```sql
postgres=# CREATE USER testuser WITH ENCRYPTED PASSWORD '12345678';
CREATE ROLE
```

* Create database:
```sql
postgres=# CREATE DATABASE testdb;
CREATE DATABASE
```

* Make `testuser` a superuser for created database:
```sql
postgres=# GRANT ALL PRIVILEGES ON DATABASE testdb TO testuser;
GRANT
```
:pushpin: Also don't forget to grant privileges for this user for tables and sequences in default `public` schema (if you don't have any other schemas yet, ofcourse):
```sql
postgres=# GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO testuser;
GRANT
postgres=# GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO testuser;
GRANT
```

`FIN!`
