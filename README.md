# Reproduce pgloader problem

This project is created to reproduce a problem where pgloader doesn't migrate default values for boolean columns, when migrating from MySQL to Postgres.

Note that this project uses MySQL 5.7, which is what I need to migrate from.
I tried running MySQL 8.0, to see if the problem persisted there,
but hit the well-known [QMYND:MYSQL-UNSUPPORTED-AUTHENTICATION](https://github.com/dimitri/pgloader/issues/782) problem,
and bailed since 8.0 wasn't a strict requirement on my side.

## Start MySQL and PostgreSQL

Start the infrastructure:
`docker-compose up -d`

## Create MySQL Schema

Start your favourite MySQL client, and connect to the `demodb` database.
A JDBC connection string looks like: `jdbc:mysql://root:admin@localhost:3308/demodb?useSSL=false&createDatabaseIfNotExist=true`.
Inside the MySQL container, you can connect with `mysql -padmin` and then `use demodb`.

Execute the contents of `mysql-schema.sql` to create the database schema.

Check the structure with SQL statement `DESCRIBE demo_table;`.
Note that column `default_false_boolean` has default value `false`,
and column `default_one_int` has default value 1.

```
mysql> describe demo_table;
+-----------------------+---------+------+-----+---------+-------+
| Field                 | Type    | Null | Key | Default | Extra |
+-----------------------+---------+------+-----+---------+-------+
| default_boolean_false | bit(1)  | YES  |     | b'0'    |       |
| default_boolean_true  | bit(1)  | YES  |     | b'1'    |       |
| default_int_one       | int(11) | YES  |     | 1       |       |
+-----------------------+---------+------+-----+---------+-------+
```

## Migrate Using Pgloader

Start a pgloader container: `docker run --rm --name pgloader --network=pgloader-playground_platform -it dimitri/pgloader:ccl.latest /bin/bash`
(change the network name if your base folder is different from `pgloader-playground`)

The image has no editor installed, so install one that you prefer:
```shell
apt update # and ignore any errors
apt install -y vim
```
Paste the contents of `mysql-to-postgres.load` into a file in the pgloader container, then run it:
```shell
vi mysql-to-postgres.load
pgloader mysql-to-postgres.load
```

## Verify Results in PostgreSQL

Inside the Postgres container, connect to the database: `psql -U demo`
Check the structure with command `\d demo_table`.

```
demo=# \d demo_table
                      Table "public.demo_table"
        Column         |  Type   | Collation | Nullable |   Default
-----------------------+---------+-----------+----------+-------------
 default_boolean_false | boolean |           |          |
 default_boolean_true  | boolean |           |          |
 default_int_one       | bigint  |           |          | '1'::bigint
```

Note that the default values for the boolean columns are missing.
