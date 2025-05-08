# redshift-migrate-db
![Migrate-DB](images/migrate-db.png)

Redshift Database Migration Using Data Sharing
Purpose
This utility enables the migration of a single database from one Amazon Redshift cluster to another using Redshift Data Sharing, allowing seamless, secure access without the need to move or copy data manually.

Scope
User & Group Replication

Users, groups, and their memberships not already present in the target cluster will be created automatically.

Note: Redshift roles are not supported in this version.

Object Migration

The following database objects are replicated if not already present in the target:

Schemas, Tables, Primary Keys, Foreign Keys, Stored Procedures, User-Defined Functions (UDFs)

Objects not in scope include: models, datashares, and other non-listed object types.

Tables with multiple identity columns are not supported. For tables with single identity columns, the utility will convert them to "DEFAULT identity" with calculated seed values to avoid duplicates.

All schemas are migrated except those explicitly excluded via configuration.

Permissions & Ownership

Object ownership (schemas, tables, procedures, functions) is replicated from source to target.

Grants to users and groups are replicated for supported objects.

Default permissions (e.g., schema-level or user-to-user grants) are also applied.

Permissions for unsupported object types are not migrated.

Data Sharing Setup

The utility automatically configures Redshift data sharing:

Creates external schemas in the target.

Adds necessary tables to the datashare.

Cross-region and cross-account sharing are not supported in this release.

Data Loading

Data is transferred in parallel threads for performance.

Tables with interleaved sort keys are skipped (unsupported by Redshift Data Sharing).

Materialized views are recreated in the target cluster based on the source.

Standard views are created after materialized views to resolve dependency chains.

Prerequisites
An EC2 instance (Amazon Linux or CentOS) with network access to both the source and target Redshift clusters on port 5439.

SSH access to the EC2 instance to execute the migration utility.

## Linux Setup
1. Ensure you have the PostgreSQL client installed.

`sudo yum install postgresql.x86_64 -y`

2. Configure the `.pgpass` file so you can connect to the Source and Target clusters without being prompted for a password.

```
echo "Source"
echo "source.account.us-east-2.redshift.amazonaws.com:5439:*:awsuser:P@ssword1" > ~/.pgpass
echo "Target"
echo "target.account.us-east-2.redshift-serverless.amazonaws.com:5439:*:awsuser:P@ssword1" >> ~/.pgpass
echo "Security"
chmod 600 ~/.pgpass
```

In the above example, the awsuser has the password P@ssword1 for both the Source and Target clusters. Also be sure to change the cluster entries from the example to your actual cluster endpoints.  More information here on the .pgpass file: https://www.postgresql.org/docs/current/libpq-pgpass.html


3. Install the git client.

`sudo yum install git -y`

4. Clone this repository.

```
git clone https://github.com/aws-samples/redshift-migrate-db.git
cd redshift-migrate-db
```

## Script setup
1. cp `config.sh.example` to `config.sh`

2. Edit `config.sh` and make changes for your Redshift Source and Target clusters.

3. All schemas found in the Source cluster will be migrated to the Target cluster except for those exluded in the config.sh file. Edit the `EXCLUDED_SCHEMAS` variable to add additional schemas.

4. Most steps in the migration are executed in parallel and the level of parallelism is handled by the `LOAD_THREADS` variable in `config.sh`. Testing has shown the default has worked well.

5. There is automatic retry logic in the scripts in case of a failure. The script will execute again autoamtically based on the `RETRY` variable in `config.sh`. You can disable this by setting this variable value to 0. However, some scripts need the retry logic because of dependent objects that can be found in views, procedures, tables, and foreign keys. 

## Script execution
You can run the 0*.sh scripts one at a time like this:

`./01_create_users_groups.sh`

or you can use the `migrate.sh` script to run all of the scripts like this:

`./migrate.sh`

For larger migrations, you may want to run this in the background like this:

`nohup ./migrate.sh > migrate.log 2>&1 &`

You can monitor the progress by tailing the migrate.log file like this:

`tail -f migrate.log`


### Script Detail
**01_create_users_groups.sh** migrates existing users and groups to the target database. It will also add users to the groups. If the user doesn't exist in the target cluster, a default password is used in the target cluster and the password will be set to be expired. As of now, Roles are out of scope for this utility.

**02_migrate_ddl.sh** migrates schemas, tables, primary keys, foreign keys, functions, and procedures to the target database. All functions have retry logic except for create schema. This ensures objects with dependencies eventually get created.

The create_table logic is robust and uses the following logic:

```
Does target table exist? 
 ├── Yes
 │   └── Does target table have an identity column?
 │       └── Yes
 │           ├── Does target table have data in it?
 │           │   ├── No
 │           │   │   └── Does source table have data in it?
 │           │   │       └── Yes -> Get max value from identity column in source and target seed value. Are seed values diffent?
 │           │   │           ├── Yes -> Recreate target table with new seed and use default identity instead of identity.
 │           │   │           └── No -> Do nothing
 │           │   └── Yes -> Do nothing
 │           └── No -> Do nothing
 └── No
     └── Does source table have an identity column? 
         ├── Yes
         │   └── Does source table have data in it?
         │       ├── Yes -> Get max value from identity column in source, create table in target with max + 1 as seed and default identity instead of identity.
         │       └── No -> Use 1 as seed and create table with default identity instead of identity.
         └── No -> Create table in target with source DDL.
```

Note that tables with interleaved sort keys are excluded and all schemas will be migrated except those configured to be excluded. See `config.sh` for more details.

Key Scripts
03_migrate_permissions.sh
Migrates object-level and default permissions from the source cluster to the target, including:

Ownership of schemas, tables, functions, and stored procedures

Grants on schemas, tables, functions, and procedures to individual users and groups

Default permissions:

Schemas → Users

Schemas → Groups

Users → Users

Users → Groups
Note: Role permissions are not handled in this version.

04_setup_datasharing.sh
Automates the setup of Redshift Data Sharing:

Creates a datashare in the source cluster

Adds relevant schemas to the datashare

Grants access to the datashare in the target cluster

Creates external schemas in the target

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
