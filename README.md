
AWS Database Migration Service with LOB columns - Quick Guide
=============================================================

What's Here
-----------

A step by step working sample on how to migrate a table containing LOB columns 
using AWS Database Migration Service (AWS DMS), 
from a PostgreSQL database running on an AWS EC2 instance, 
to an AWS RDS PostgreSQL database. 

Main steps covered are:

-  `Set Up the Source Database <#source-database>`__
-  `Set Up the Target Database <#target-database>`__
-  `Set Up AWS DMS <#aws-dms>`_

It assumes you are familiar with:
-	AWS Management Console
-	AWS EC2 instances
-	AWS IAM
- AWS Database Migration Service (AWS DMS)

Let's get started...

### Step1: Set Up the Source Database


First, create your AWS EC2 instance from AWS Management Console and connect to it by SSH.

Install PostgreSQL by running the following commands (Ubuntu):

```sh
$ sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
$ wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
$ sudo apt-get update
$ sudo apt-get -y install postgresql-12
```

To check the installation has been successful:

```sh
$ systemctl status postgresql.service
$ systemctl status postgresql@12-main.service
$ systemctl is-enabled postgresql
```

Change the password for your `postgres` user:

```sh
$ sudo -u postgres psql
$ postgres=#\password
```

Apply the following configuration changes:

```sh
# Edit pg_hba.conf in vim
sudo vim /etc/postgresql/12/main/pg_hba.conf

# Near bottom of file after local rules, add rule (allows remote access):
host    all             all             0.0.0.0/0               md5

# save the file

# Edit config in vim
sudo vim /etc/postgresql/12/main/postgresql.conf

# Change listen_address to listen to external requests:
listen_address='*'
  
# save the file
```

As per AWS documentation, apply the configuration changes documented at
[Prerequisites for using a PostgreSQL database as a source for AWS DMS](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Source.PostgreSQL.html#CHAP_Source.PostgreSQL.Prerequisites)
as well.

For your reference, find both files in `./source_config` folder.

Restart Postgres database:

```sh
$ sudo /etc/init.d/postgresql restart
```

#### Now let's create and prepare the database...

From within your EC2 instance:

```sh
$ sudo su postgres
$ psql
$ postgres=# CREATE DATABASE database_name;
$ postgres=# CREATE USER username with encrypted password 'user_password';
$ postgres=# GRANT ALL PRIVILEGES ON DATABASE database_name to username;
```

> Replace `database_name`, `username` and `user_password` accordingly.
>
> You will need these values later, when setting up AWS DMS source endpoint.

Copy (sftp) the content of `./data` folder into your EC2 instance  and 
grant read access to `postgres` user.

> _For this working sample, I copied them to `/home/ubuntu/db-migration/data/`_

Now, create the source table and import the files:

```sh
$ sudo su postgres

$ psql database_name

-- create the document table
$ CREATE TABLE document(filename text);

-- load the file names from disk to the document table
$ COPY document FROM PROGRAM 'ls /home/ubuntu/db-migration/data/* -R' WITH (format 'csv');

-- add the following additional columns
$ ALTER TABLE document ADD COLUMN content bytea, ADD COLUMN content_oid oid;

-- add the document to large object storage and return the link id
$ UPDATE document SET content_oid = lo_import(filename);

-- pull document from large object storage
$ UPDATE document SET content = lo_get(content_oid);

-- delete the files from large object storage
$ SELECT lo_unlink(content_oid) FROM document;

-- add the PK to your table
$ ALTER TABLE document ADD COLUMN id serial;

$ ALTER TABLE document ADD PRIMARY KEY (id);
```


For the migration to be successful, the target table must have a 
PK and the LOB column(s) must be ‘nullable’.

If the target table does not exist yet, your source table must 
have a PK and the type of the PK column cannot be of a type that AWS DMS treats as LOB.

As an example, if you use `filename` column as PK, AWS will treat that column as a 
LOB during the migration because its type is `text`. 
Hence, AWS DMS will create `filename` column as ‘nullable’ in the target table 
and consequently your target table will not have a PK.

If the target table exists in the target database, ensure you have the PK constraint already in place. 
In addition, ensure the LOB column is marked as “nullable”.


### Step 2: Set Up the Target Database


To set up the PostgreSQL database follow the tutorial 
[Create and Connect to a PostgreSQL database with Amazon RDS](https://aws.amazon.com/getting-started/tutorials/create-connect-postgresql-db/).


### Step 3: Set Up AWS DMS


1/ Create a Replication Instance in AWS DMS

2/ Create Source and Target Endpoints for your database migration

3/ Create a Replication Task in AWS DMS.

Select `Full LOB mode` on `Task settings` section so all LOBs are migrated 
from source to target regardless of their size.

> When a task is configured to use `full LOB mode`, AWS DMS retrieves LOBs in pieces.
> The `LOB chunk size` option determines the size of each piece. 
> When setting this option, pay particular attention to the maximum packet size allowed 
> by your network configuration. 
> If the LOB chunk size exceeds your maximum allowed packet size, you might see disconnect errors.

Leave the default value as 64(K).

4/ Start the replication task and wait until the status is `Load complete`


> If you can’t find AWS DMS logs in CloudWatch, check the role `dms-cloudwatch-logs-role` 
exists in IAM.

>If it does not, follow the instructions at 
[Why can't I see CloudWatch Logs for an AWS DMS task?](https://aws.amazon.com/premiumsupport/knowledge-center/dms-cloudwatch-logs-not-appearing/) 
to enable AWS DMS logs. 









