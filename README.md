[![Build](https://github.com/hashgraph/hedera-mirror-node/actions/workflows/importer.yml/badge.svg)](https://github.com/hashgraph/hedera-mirror-node/actions)
[![codecov](https://img.shields.io/codecov/c/github/hashgraph/hedera-mirror-node/master)](https://codecov.io/gh/hashgraph/hedera-mirror-node)
[![GitHub](https://img.shields.io/github/license/hashgraph/hedera-mirror-node)](LICENSE)
[![Discord](https://img.shields.io/badge/discord-join%20chat-blue.svg)](https://hedera.com/discord)

# Hedera Mirror Node

Hedera Mirror Node exposes Hedera Hashgraph transactions, transaction records, account balances, and events generated by
the Hedera mainnet (or testnet, if so configured) via a REST & gRPC API.

## Overview

Hedera mirror nodes receive the information from the mainnet nodes and they provide value-added services such as
providing audit support, access to historical data, transaction analytics, visibility services, security threat
modeling, data monetization services, etc. Mirror nodes can also run additional business logic to support applications
built using Hedera mainnet.

While mirror nodes receive information from the mainnet nodes, they do not contribute to consensus on the mainnet, and
their votes are not counted. Only the votes from the mainnet nodes are counted for determining consensus. The trust of
Hedera mainnet is derived based on the the consensus reached by the mainnet nodes. That trust is transferred to the
mirror nodes using cryptographic signatures on a chain of records (account balances, events, transactions, etc).

## Beta Mirror Node

Eventually, the mirror nodes can run the same code as the Hedera mainnet nodes so that they can see the transactions in
real time. To make the initial deployments easier, the beta mirror node strives to take away the burden of running a
full Hedera node through creation of periodic files that contain processed information (such as account balances or
transaction records), and have the full trust of the Hedera mainnet nodes. The beta mirror node software reduces the
processing burden by receiving pre-constructed files from the mainnet, validating those, populating a database and
providing REST APIs.

### Advantages

- Lower compute, bandwidth requirement
- It allows users to only save what they care about, and discard what they don’t (lower storage requirement)
- Easy searchable database so the users can add value quickly
- Easy to consume REST APIs to make integrations faster

### Description

The beta mirror node works as follows:

- When a transaction reaches consensus, Hedera nodes add the transaction and its associated record to a record file.
- The file is closed on a regular cadence and a new file is created for the next batch of transactions and records. The
  interval is currently set to 2 seconds but may vary between networks.
- Once the file is closed, nodes generate a signature file which contains the signature generated by the node for the
  record file.
- Record files also contain the hash of the previous record file, thus creating an unbreakable validation chain.

- The signature and record files are then uploaded from the nodes to Amazon S3 and Google File Storage.

- This mirror node software downloads signature files from either S3 or Google File Storage.
- The signature files are validated to ensure at least 1/3 of the nodes in the address book (stored in a `0.0.102` file)
  have the same signature.
- For each valid signature file, the corresponding record file is then downloaded from the cloud.
- Record files can then be processed and transactions and records processed for long term storage.

- Event files are handled in same manner as record files.

- In addition, nodes regularly generate a balance file which contains the list of Hedera accounts and their
  corresponding hbar balance and miscellaneous token balances which is also uploaded to S3 and Google File Storage.
- The files are also signed by the nodes.
- This mirror node software can download the balance files, validate at least 1/3 of nodes have signed and then process
  the balance files for long term storage.

### Rosetta API

In addition to the REST API which exposes the persisted network entities (accounts, balances, transactions, topics and
tokens), the Mirror Node also provides a [Rosetta API](https://www.rosetta-api.org/docs/welcome.html) compliant REST API
server. This exposes a subset of data with a focus on blockchain data integration.
See [rosetta-server](docs/rosetta-server.md) for more details.

## Getting Started

### Technologies

Multiple technologies are utilized in the mirror node. The following topics are areas where basic knowledge is valuable
to understanding the mirror node operations.

- [Git](https://git-scm.com/about)
- [Maven](https://maven.apache.org/guides/getting-started/index.html)
- [Docker](https://docs.docker.com/)
  - [docker-compose commands](https://docs.docker.com/compose/reference/overview/)
  - [docker commands](https://docs.docker.com/engine/reference/commandline/docker/)
- [NodeJS](https://nodejs.org/en/about/)
- [Spring](https://spring.io/quickstart)
  - [Spring Boot](https://docs.spring.io/spring-boot/docs/current/reference/html/getting-started.html#getting-started)
  - [Externalized Configurations](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-external-config)
- [PostgreSQL](https://www.postgresql.org/docs/9.6/index.html)
  - [SQL commands](https://www.postgresql.org/docs/9.6/sql-commands.html)
  - [Client Application & Utilities](https://www.postgresql.org/docs/9.6/reference-client.html)
- [Go](https://golang.org/)

### Prerequisite Tools

Ensure these tools are installed prior to running the mirror node

- [Docker](https://www.docker.com/products/docker-desktop)

### Running Mirror Node

To run the mirror node, execute these 3 commands in your terminal.

```bash
git clone https://github.com/hashgraph/hedera-mirror-node.git
cd hedera-mirror-node
docker-compose up
```

> **_NOTE:_** This defaults to a bucket setup for demonstration purposes. See the [Public Networks](#public-networks)
> section for how to configure the mirror node to retrieve production data.

## Data Access

### Demo

The free option utilizes a bucket setup for demonstration purposes. This is the default option and requires no
additional steps.

### Public Networks

To access real data from the testnet or mainnet buckets the requester pays flow
for [GCP](https://cloud.google.com/storage/docs/requester-pays)
or [AWS](https://docs.aws.amazon.com/AmazonS3/latest/dev/RequesterPaysBuckets.html) must be utilized. Here the charges
associated with request and download of transaction data are paid for by the requester and not the bucket owner.

To achieve this, two simple steps must be taken to configure the mirror node:

- Uncomment the contents of [application.yml](./application.yml) file under the root folder
- Customize the contents according to your setup. See [configurations](docs/configuration.md) for a detailed description
  of the configuration options.

> **_Note_** The [application.yml](./application.yml) file contents represent the minimal set of fields required to
> configure requester pays and must all be uncommented and filled in. The file is referenced in the
> [docker-compose.yml](docker-compose.yml) and allows customized configuration for each of the sub modules.

See the [Docker Compose](/docs/installation.md#running-via-docker-compose) documentation for further details.

### Verify Operational Mirror Node

When running the Mirror Node using Docker, activity logs and container status for each module container can be viewed in
the [Docker Desktop Dashboard](https://docs.docker.com/desktop/dashboard/) to verify expected operation.

You can also interact with some module containers (gRPC and REST API's) to verify their operation.

First list running docker container information using:

     docker ps

Useful information for all the running containers such as CONTAINER ID, STATUS, IP's and ports will be displayed as
below:

    CONTAINER ID    IMAGE                                           COMMAND                 CREATED         STATUS          PORTS                   NAMES
    21fa2a986d99    gcr.io/mirrornode/hedera-mirror-rest:0.12.0     "docker-entrypoint.s…"  7 minutes ago   Up 12 seconds   0.0.0.0:5551->5551/tcp  hedera-mirror-node_rest_1
    56647c384d49    gcr.io/mirrornode/hedera-mirror-grpc:0.12.0     "java -cp /app/resou…"  8 minutes ago   Up 16 seconds   0.0.0.0:5600->5600/tcp  edera-mirror-node_grpc_1

An 'Up' STATUS confirms the mirror node sub modules are running.

Using the IP and port, the gRPC and REST API's endpoints can be called to confirm data is processed and available.

Using the CONTAINER ID docker containers can be accessed for in depth troubleshooting.

#### Importer

Logs similar to the following snippets can be used to confirm the Importer is downloading and parsing transactions to
the database

    Current version of schema "public": << Empty Schema >>
    ...
    Migrating schema "public" to version 1.0 - Init
    ...
    Successfully applied 41 migrations to schema "public" (execution time 00:01.656s)
    ...
    Loading record format version 2 from record file: ...
    Inserted 2 transactions, 6 transfer lists, 0 files, 0 contracts, 0 claims, 0 topic messages, 0 non-fee transfers
    Finished parsing 2 transactions from record file ....

#### Database

The following log can be used to confirm the database is up and running:

    LOG: database system is ready to accept connections

If you have postgresql installed you can connect directly to your database using the following psql command and
default [configurations](docs/configuration.md):

    psql "dbname=mirror_node host=localhost user=mirror_node password=mirror_node_pass port=5432"

If psql is not available you can `docker exec -it <CONTAINER ID> bash` into the db container and run the same command
above

Some useful basic queries to help view database contents and data include

```
\du
select * from account_balance limit 5;
select * from transactions limit 5;
select * from topic_message limit 5;
```

#### REST API

The REST API container will display logs similar to the below at start:

    Server running on port: 5551

To manually verify REST API endpoints follow the [Operations](docs/operations.md#verifying-1) details.

#### gRPC API

The gRPC container will display logs similar to the below at start

    MirrorGrpcApplication Started MirrorGrpcApplication ....
    Listener Starting to poll every 1000ms

To manually verify the gRPC streaming endpoint follow the [Operations](docs/operations.md#verifying) details.

Additionally, logs of each module container can be viewed to verify expected operation or decipher issues.
See [Troubleshooting](docs/troubleshooting.md) for details.

Simply access the terminal on each container with the following command and refer to the [perations](docs/operations.md)
documentation for directions on where and how to view logs.

     docker exec -it <CONTAINER ID> bash

> **_Note_** You cannot exec into the gRPC, Monitor and Importer containers as they are
> [distroless](https://github.com/GoogleContainerTools/distroless) java images.

#### Rosetta API

The Rosetta API container will display logs similar to the below at start:

    Successfully connected to Database
    Serving Rosetta API in ONLINE mode
    Listening on port 5700

To manually verify the Rosetta API endpoints follow the [Operations](docs/operations.md#verifying-2) details.

## Documentation

- [Installation](docs/installation.md)
- [Configuration](docs/configuration.md)
- [Operations](docs/operations.md)
- [Testing](docs/testing.md)
- [Troubleshooting](docs/troubleshooting.md)

## Releasing

To prepare for a new release, first update `release.version` and `release.chartVersion` in the root `pom.xml`. Then run:

```
./mvnw clean package -N -P=release
helm dependency update charts/hedera-mirror
```

## Contributing

Contributions are welcome. Please see the [contributing](CONTRIBUTING.md) guide to see how you can get involved.

## Code of Conduct

This project is governed by the [Contributor Covenant Code of Conduct](CODE_OF_CONDUCT.md). By participating, you are
expected to uphold this code of conduct. Please report unacceptable behavior to [oss@hedera.com](mailto:oss@hedera.com)

## License

[Apache License 2.0](LICENSE)
