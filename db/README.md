# Schema
[Schema](../backend/db)

# Connecting to the database -- Development & Staging

Staging is done on primarily in gsa-open-ops cloud.gov space. It is a postgresql on AWS RDS.

Using the command line, you can "bind" an app with a service. The hosted backend should be "bound" with the hosted db.

### Step 1: Log in and point your target to the sandbox.

Refer to the [docs](https://cloud.gov/docs/apps/managed-services/)

### Step 2: Create a service key

You will need a service key for the backend to connect to the DB

You can check if the API call to create the db has been completed by looking at the `last operation`, which should say "create succeeded":

> cf services

Just because the `create db` command was run doesn't mean the service has finished provisioning. Try to create a key to verify:

> cf create-service-key smeqa-db smeqa-db-service-key

Note: You may need to wait a few minutes before this command works.

### Step 3: Use credentials to connect locally running backend to sandbox

1. Copy/paste the contents of `backend/template-env.js` into a new file, `backend/.env`.

2. Use the service key to get your credentials by running:

   > cf service-key smeqa-db smeqa-db-service-key

3. Copy the credentials to your newly created `.env` file

### Step 4: Set-up ssh-port forwarding

Your local backend will try to connect to the cloud.gov DB directly over port 5432. Unfortunately, direct connections to the database do not work unless you're in the same space as the DB.

To work around this, we need to setup a "host app" that uses ssh port-forwarding to communicate with the DB. Although we can create a dedicated host app for this purpose, we can use the existing `smeqa` cloud.gov app as the host.

More details are here:
https://docs.cloudfoundry.org/devguide/deploy-apps/ssh-services.html

In a separate terminal window, run the following:




```
// login
> cf login -a api.fr.cloud.gov --sso

// get the guid of the smeqa app
// this host app that tunnels our request to the db
> myapp_guid=$(cf app --guid smeqa-staging)

// define the db host and port
// this is where the host app should forward requests to.
// Call this destination `tunnel`
myapp_guid=$(cf app --guid smeqa-staging); tunnel=$(cf curl /v2/apps/$myapp_guid/env | jq -r '[.system_env_json.VCAP_SERVICES."aws-rds"[0].credentials | .host, .port] | join(":")'); cf ssh -N -L 5432:$tunnel smeqa-staging

// actually do the port forwarding
// this tells the host app to auto-forward incoming requests on port 5432 to the the db at the `tunnel` path defined above. The db is also happening to listen to port 5432
cf ssh -N -L 5432:$tunnel smeqa-staging
```

TODO: add this to a `prestart` script

# Managing the Sandbox DB

To wipe all data in the sandbox DB and repopulate it with staging data, run the following script:

> yarn reset-db

This deletes all tables, recreates them, and populates it with fake data.

TODO: create fake data to populate database

# Setting up the Sandbox Database

You should only have to do this once

Most of the setup steps below references the documentation:
https://cloud.gov/docs/services/relational-database/

### Step 1: Log in and point your target to the sandbox. Refer to the [docs](https://cloud.gov/docs/apps/managed-services/)

### Step 2: Create the DB service in the sandbox space

> cf create-service aws-rds shared-psql smeqa-sandbox-db -c '{"storage": 1}'

We're only specifying 1GB of space since we don't need a lot for the sandbox version. This can change in production, but should be small nonetheless

For production, there is a different aws-rds plan to specify. See the [list of plans](https://cloud.gov/docs/services/relational-database/) for options and configuration parameters

### Step 3: Connect the hosted backend to the sandbox db

#### Step 3a. (optional) Manually bind the backend with the db

The backend app would need credentials to read/write to the DB service. Passing credentials "binds" and app with a service. There are two ways of binding the app:

> cf bind-service smeqa-backend smeqa-db

Note: At this point, your target should be the sandbox space, though this same command would work in the production space

#### Step 3b. Update the manifest.yml for automatic binding during deploys

Instead of 3a, you an add the db service in the backend's `manifest.yml` to automatically bind the service whenever you push the backend.

```
...
services:
 - smeqa-db
```

Binding a service creates a `DATABASE_URL` environment variable for the backend app, which contains the the credentials to connect to the db.

#### Step 3c. (optional) Verify the backend and db have been connected via ssh

There are a few ways, but the easiest is to use this tool:
https://github.com/18F/cf-service-connect#readme

> cf connect-to-service smeqa smeqa-db

### Step 4: Setup your locally running backend to connect to the sandbox

You will need a service key for the backend to connect to the DB

You can check if the API call to create the db has been completed by looking at the `last operation`, which should say "create succeeded":

> cf services

Just because the `create db` command was run doesn't mean the service has finished provisioning. Try to create a key to verify:

> cf create-service-key smeqa-db smeqa-db-service-key

Note: You may need to wait a few minutes before this command works.

After the key is created, you need to get your credentials from it:

> cf service-key smeqa-db smeqa-db-service-key

### Step 5: Use credentials to connect locally running backend to sandbox

Copy the credentials created above to your environment variables file at `backend/.env`

### Step 6: Set-up ssh-port forwarding

Your local backend will try to connect to the cloud.gov db directly over port 5432. Unfortunately, direct connections to the database do not work unless you're in the same space as the DB.

To work around this, we need to setup a "host app" that uses ssh port-forwarding to communicate with the db. Although we can create a dedicated host app for this purpose, we can use the existing `smeqa` cloud.gov app as the host.

More details are here:
https://docs.cloudfoundry.org/devguide/deploy-apps/ssh-services.html

```
// get the guid of the smeqa app
// this host app that tunnels our request to the db
> myapp_guid=$(cf app --guid smeqa)

// define the db host and port
// this is where the host app should forward requests to.
// Call this destination `tunnel`
tunnel=$(cf curl /v2/apps/$myapp_guid/env | jq -r '[.system_env_json.VCAP_SERVICES."aws-rds"[0].credentials | .host, .port] | join(":")')

// actually do the port forwarding
// this tells the host app to auto-forward incoming requests on port 5432 to the the db at the `tunnel` path defined above. The db is also happening to listen to port 5432
cf ssh -N -L 5432:$tunnel smeqa
```

=============================================================================================================================

Instructions for targeting the `deployProd.yml` manifest file.

Assumes an application called `smeqa-rr-tool` and a database service named `smeqa-stage-db` per the manifest file.

https://cloud.gov/docs/services/relational-database/
create the database service, note the name

- https://login.fr.cloud.gov/passcode
get code, save it for login step
- cf login -a api.fr.cloud.gov  --sso
enter code from previous step
- cf services
sample output:
```
Getting service instances in org sandbox-gsa / space first.last as first.last@gsa.gov...

name             offering   plan         bound apps      last operation     broker       upgrade available
smeqa-stage-db   aws-rds    small-psql   smeqa-rr-tool   create succeeded   aws-broker   no
```
- cf bind-security-group trusted_local_networks_egress sandbox-gsa --space first.last
sample output:
```Assigning running security group trusted_local_networks_egress to space first.last in org sandbox-gsa as first.last@gsa.gov...
OK

TIP: If Dynamic ASG's are enabled, changes will automatically apply for running and staging applications. Otherwise, changes will require an app restart (for running) or restage (for staging) to apply to existing applications.```
- cf connect-to-service smeqa-rr-tool smeqa-stage-db
see `deployProd.yml` manifest file for service names to connect.
sample output for successful connection:
```Finding the service instance details...
Setting up SSH tunnel...
SSH tunnel created.
Connecting client...
psql (14.9 (Homebrew), server 15.4)
WARNING: psql major version 14, server major version 15.
         Some psql features might not work.
SSL connection (protocol: TLSv1.2, cipher: ECDHE-RSA-AES256-GCM-SHA384, bits: 256, compression: off)
Type "help" for help.

cgawsbrokerprod<XYZXYZ>=>
```
exit out
⇒  cf create-service-key smeqa-stage-db smeqa-stage-db-service-key
Creating service key smeqa-stage-db-service-key for service instance smeqa-stage-db as first.last@gsa.gov...
OK
⇒  cf service-key smeqa-stage-db smeqa-stage-db-service-key
Getting key smeqa-stage-db-service-key for service instance smeqa-stage-db as first.last@gsa.gov...

{
  "credentials": {
    "db_name": "<DBNAME>",
    "host": "cg-aws-broker-prod<XYZXYZ>.<ABCABC>.us-gov-west-1.rds.amazonaws.com",
    "name": "<DBNAME>",
    "password": "<DBPASSWORD>",
    "port": "5432",
    "uri": "postgres://<DBUSERNAME>:<DBPASSWORD>@cg-aws-broker-prod<XYZXYZ>.<ABCABC>.us-gov-west-1.rds.amazonaws.com:5432/<DBNAME>",
    "username": "<DBUSERNAME>"
  }
}
⇒  cf enable-ssh smeqa-rr-tool
Enabling ssh support for app smeqa-rr-tool as first.last@gsa.gov...
ssh support for app 'smeqa-rr-tool' is already enabled.
OK

TIP: An app restart may be required for the change to take effect.
$ cf ssh -L <LOCALPORT>:cg-aws-broker-prod<XYZXYZ>.<ABCABC>.us-gov-west-1.rds.amazonaws.com:5432 smeqa-rr-tool
This should result in an SSH session on the remote app command line.

Your database should now be available on local port <LOCALPORT> with the host, database, and username and password credentials from the service key step.

- Create the database
- run the migrations in `/db/migrations` in order

current output:
```
{"message":"no pg_hba.conf entry for host \"10.10.2.8\", user \"<DBUSERNAME>\", database \"cgawsbrokerprod<XYZXYZ>\", no encryption"}
```
