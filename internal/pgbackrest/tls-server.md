<!--
 Copyright 2021 Crunchy Data Solutions, Inc.
 Licensed under the Apache License, Version 2.0 (the "License");
 you may not use this file except in compliance with the License.
 You may obtain a copy of the License at

 http://www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License.
-->

# pgBackRest TLS Server

A handful of pgBackRest features require connectivity between `pgbackrest` processes
on different pods:

- [dedicated repository host](https://pgbackrest.org/user-guide.html#repo-host)
- [backup from standby](https://pgbackrest.org/user-guide.html#standby-backup)

When a PostgresCluster is configured to store backups on a PVC, we start a dedicated
repository host to make that PVC available to all PostgreSQL instances in the cluster.

The repository host runs a `pgbackrest` server that is secured through TLS and
[certificates][]. When performing backups, it connects to `pgbackrest` servers
running on PostgreSQL instances (as sidecars). Restore jobs connect to the
repository host to fetch files. PostgreSQL calls `pgbackrest` which connects
to the repository host to [send and receive WAL files][archiving].

[archiving]: https://www.postgresql.org/docs/current/continuous-archiving.html
[certificates]: certificates.md


The `pgbackrest` command acts as a TLS client and connects to a pgBackRest TLS
server when `pg-host-type=tls` and/or `repo-host-type=tls`. The default for these is `ssh`:

- https://github.com/pgbackrest/pgbackrest/blob/release/2.36/src/config/parse.auto.c#L3580
- https://github.com/pgbackrest/pgbackrest/blob/release/2.36/src/config/parse.auto.c#L5941


The pgBackRest TLS server is configured through the `tls-server-*` [options](config.md).
In pgBackRest 2.36, changing any of these options or changing certificate contents
requires a restart of the server.

- `tls-server-address`, `tls-server-port` <br/>
  The network address and port on which to listen. pgBackRest 2.36 listens on
  the *first* address returned by `getaddrinfo()`. There is no way to listen on
  all interfaces.

  - https://github.com/pgbackrest/pgbackrest/blob/release/2.36/src/common/io/socket/server.c#L169
  - https://github.com/pgbackrest/pgbackrest/blob/release/2.36/src/common/io/socket/common.c#L87

- `tls-server-cert-file`, `tls-server-key-file` <br/>
  The [certificate chain][certificates] and private key pair used to encrypt connections.

- `tls-server-ca-file` <br/>
  The certificate used to verify client [certificates][].
  [Required](https://github.com/pgbackrest/pgbackrest/blob/release/2.36/src/config/parse.auto.c#L8487).

- `tls-server-auth` <br/>
  A map/hash/dictionary of certificate common names and the stanzas they are authorized
  to interact with.
  [Required](https://github.com/pgbackrest/pgbackrest/blob/release/2.36/src/config/parse.auto.c#L8471).


In pgBackRest 2.36, sending SIGHUP, SIGINT, or SIGTERM to the TLS server all
cause it to exit with code 63, TermError.

- https://github.com/pgbackrest/pgbackrest/blob/release/2.36/src/common/exit.c#L73-L75
- https://github.com/pgbackrest/pgbackrest/blob/release/2.36/src/common/exit.c#L62
- https://github.com/pgbackrest/pgbackrest/blob/release/2.36/src/common/error.auto.c#L48

```
P00   INFO: server-start command end: terminated on signal [SIGHUP]
P00   INFO: server-start command end: terminated on signal [SIGINT]
P00   INFO: server-start command end: terminated on signal [SIGTERM]
```
