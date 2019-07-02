# PgBouncer Overview

[PgBouncer](https://pgbouncer.github.io/) is an open-source, lightweight, single-binary connection-pooling middleware for PostgreSQL. PgBouncer maintains a pool of connections for each locally stored user-database pair. It is typically configured to hand out one of these connections to a new incoming client connection, and return it back in to the pool when the client disconnects. PgBouncer can pool connections to one or more PostgreSQL databases on possibly different servers and serve clients over TCP and Unix domain sockets. For a more hands-on experience, see this brief [tutorial on how to create a PgBouncer](https://pgdash.io/blog/pgbouncer-connection-pool.html) for PostgreSQL database.

With connection pooling, the clients connect to a proxy server which maintains a pool of direct connections to the real PostgreSQL server. We are palnning to introduce a CRD based implementation of PgBouncer baked right into KubeDB for Postgres. We are hoping that the implementation of connection pooling will be effective in improving the performance of your apps by decreasing the load on your PostgreSQL servers.

## Goal

 Goal of this project is to introduce the following features to KubeDB:

- A full fledged yet separate connection pooling Middleware for Postgres
- Finer control over every bit of configuration via CRD manipulation
- Quick and easy access to PgBouncer admin interface, logs, and stats
- Possibility of stats management with Prometheus

## PgBouncer CRD

Users will be able to manage connection to one or more Postgres databases of their choice using our PgBouncer implementation. The following is an example of a proposed PgBouncer in action.

**what users need to do**

- Create a PgBouncer custom resource.

A sample CRD to create a PgBouncer for KubeDB managed PostgreSQL databases 

```yaml
apiVersion: kubedb.com/v1alpha1
kind: Pgbouncer
metadata:
  name: pgbouncer
  namespace: demo
spec:
  databases:
  - dbAlias: "any-unique-alias" 
    dbName: "name-of-an-existing-pg-database" //default: "postgres"
    pgObjectName: "name-of-the-postgres-object-containing-the-db"   
  - dbAlias: "another-unique-alias"
    dbName: "name-of-another-existing-pg-database"
    pgObjectName: "name-of-the-postgres-object-containing-the-mentioned-db"   
  pgBouncerConfig:
    poolMode: "session"
    maxClientConn: 10000
    defaultPoolSize: 100
    autodbIdleTimeout: 3600
    minPoolSize: 100
    maxDbConnections: 100
    maxUserConnections: 20
    adminUsers:
    - admin1
    - admin2
  users:
  - username: user1
    password: user1Pass
  - username: user2
    password: user2Pass
  - username: admin1
    password: admin1Pass
  - username: admin2
    password: admin2Pass
   
```

Here, `Pgbouncer` will be introduced as a new `kind` to handle connection pooling for PostgreSQL. 

The `spec` section of `Pgbouncer` will have three fields to specify the database info, pgbouncer connection configuration, and user list. 

#### spec.databases:

This field is to be used to specify detailed information about PostgreSQL databases for which users want to use connection pooling.

- `spec.databases.dbAlias` is used to name a server-database pair. This alias is used to access the mentioned database  via pgbouncer. 
- `spec.databases.dbName` specifies the name of the target database. This must be a valid database name to facilitate a successful connection. 
- `spec.databases.pgObjectName` specifies the name of the Postgres object in which the target database is running. This name is used to establish connection from PgBouncer's proxy server to the target server. Primary pod of the Postgres object or the associated service is to be used to establish the said connection. 

#### spec.pgBouncerConfig:

This field is to be used to specify connection information to configure PgBouncer.

- `spec.pgBouncerConfig.poolMode` specifies when a server connection can be reused by other clients. There are three different pool modes:

  - session

    Server is released back to pool after client disconnects. This is the default mode if the field is empty.

  - transaction

    Server is released back to pool after transaction finishes.

  - statement

    Server is released back to pool after query finishes. Long transactions spanning multiple statements are disallowed in this mode.

- `spec.pgBouncerConfig.maxClientConn` specifies the maximum number of allowed client connections. Theoretical maximum used is:

  ```
  max_client_conn + (max pool_size * total databases * total users)
  ```

  if each user connects under its own username to server. If a database user is specified in connect string (all users connect under same username), the theoretical maximum is:

  ```
  max_client_conn + (max pool_size * total databases)
  ```

  The theoretical maximum should be never reached, unless somebody deliberately crafts special load for it. Still, it means you should set the number of file descriptors to a safely high number.

  The default is 100 connections.

- `spec.pgBouncerConfig.defaultPoolSize` specifies the number of server connections to allow per user/database pair. Can be overridden in the per-database configuration.

  The default is 20.

- `spec.pgBouncerConfig.minPoolSize` specifies the minimum number of server connections to be added to the pool. More server connections are added to the pool if below this number. Improves behavior when usual load comes suddenly back after period of total inactivity.

  The default is 0 (disabled).

- `spec.pgBouncerConfig.maxDbConnections` specifies the maximum number of server connections per-database (regardless of pool - i.e. user). It should be noted that when you hit the limit, closing a client connection to one pool will not immediately allow a server connection to be established for another pool, because the server connection for the first pool is still open. Once the server connection closes (due to idle timeout), a new server connection will immediately be opened for the waiting pool.

  The default is unlimited.

- `spec.pgBouncerConfig.maxUserConnections` specifies the maximum number of server connections per-user. It should be noted that when you hit the limit, closing a client connection to one pool will not immediately allow a server connection to be established for another pool, because the server connection for the first pool is still open. Once the server connection closes (due to idle timeout), a new server connection will immediately be opened for the waiting pool.

- `spec.pgBouncerConfig.adminUsers` specifies the list of users who can access PgBouncer admin console.

#### spec.users:

Specifies the list of users that are allowed to access Postgres databases via PgBouncer.

- `spec.users.username` specifies a unique username that has already been defined to be used in a target database. This name must exist in the target database that users want o connect to.  By deafult every Postgres object has a user named `postgres`.
- `spec.users.password` specifies a password to be associated with a certain username. This password can be random, and doesn't have to be the exact password associated with the username in the target database. However, users will need to use this to connect to the target database via PgBouncer.  

**How it will work?**
PgBouncer needs a configuration file and a user credentials file. The configuration file contains information about database, and connection configuration. The user credentials file contains username and password of users who are allowed to access databases via PgBouncer.
We intent to use the CRD above to collect the necessary information to fill out these information. Also the `spec.databases.pgObjectName` of the CRD is to be used in order to establish connection between the primary pod of the postgres object and the clients via PgBouncer.  

## KubeDB integration

Our implementation of PgBouncer is expected have close ties with Postgres CRD and KubeDB. As PgBouncer is not version bound, any database inside an accessible Postgres object should be accessible through PgBouncer as it works merely as a proxy between client and PostgreSQL databases. 

PgBouncer is a standalone relaying service. A single instance of PgBouncer can access many different databases running inside different Postgres objects. Therefore, the relation between PgBouncer and Postgres objects is one-to-many. Our intention is to let users leverage this feature to make their workflow easier, and more manageable. 

Additionally, PgBouncer may also possibly be created from Postgres CRD by adding PgBouncer template to it, so that the users can create a connection pooler right from the get go, and then create databases and database users later.
