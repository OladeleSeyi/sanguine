---
sidebar_position: 0
sidebar_label: API
---

# RFQ API

:::note

This guide is mostly meant for developers who are working on building their own quoter or frontend for rfq. If you are just looking to run a relayer, please see [Relayer](../Relayer). If you are looking to integrate rfq, please see the API docs below the dropdown.

:::

The RFQ API is an off-chain service that allows market makers to post quotes for certain bridge routes & tokens. Users can then read these quotes and take the liquidity by submitting a transaction on chain through the [Fast Bridge Contract](https://vercel-rfq-docs.vercel.app/contracts/FastBridge.sol/contract.FastBridge.html).

Solvers are responsible for keeping quotes fresh and implementations of the RFQ relayer should update quotes as often as possible. By default, the canonical [relayer](../Relayer) continously updates quotes by checking on-chain balances and in-flight requests and other implementations should take a similiar approach.

The implementation of the rfq api can be found [here](https://github.com/synapsecns/sanguine/tree/master/services/rfq/api). Please note that end-users and solvers will not need to run their own version of the API.


## Integrating the API

The RFQ API is a RESTful API that allows users to post quotes and read quotes. The API is incredibly simple and only has one endpoint with two methods:

- [`PUT /quotes`](./upsert-quote.api.mdx) (authenticated) - Upsert a quote
- [`GET /quotes`](./get-quotes.api.mdx) (unauthenticated) - Get all quotes, can be filtered by different parameters.

Only Solvers should be writing to the API, end-users need only read from the `/quotes` endpoint.

**API Version Changes**

An http response header "X-Api-Version" will be returned on each call response.

Any systems that integrate with the API should use this header to detect version changes and perform appropriate follow-up actions & alerts.

Upon a version change, [versions.go](https://github.com/synapsecns/sanguine/blob/master/services/rfq/api/rest/versions.go) can be referred to for further detail on the version including deprecation alerts, etc.

Please note, while Synapse may choose to take additional steps to alert & advise on API changes through other communication channels, it will remain the responsibility of the API users & integrators to set up their own detection & notifications of version changes as they use these endpoints. Likewise, it will be their responsibility review the versions.go file, to research & understand how any changes may affect their integration, and to implement any necessary adjustments resulting from the API changes.

**Authentication**

In accordance with [EIP-191](https://eips.ethereum.org/EIPS/eip-191), the RFQ API requires a signature to be sent with each request. The signature should be generated by the user's wallet and should be a valid signature of the message `rfq-api` with the user's private key. The signature should be sent in the `Authorization` header of the request. We provide a client stub/example implementation in go [here](https://pkg.go.dev/github.com/synapsecns/sanguine/services/rfq@v0.13.3/api/client).

:::note

The RFQ API expects the signatures to have V values as 0/1 rather than 27/28. The fastest way to fix this is to modify V by subtracting 27

:::


### API Urls

 - Mainnet: `rfq-api.omnirpc.io`
 - Testnet: `rfq-api-testnet.omnirpc.io`


## Running the API:

Users and relayers **are not** expected to run their own version of the RFQ API. The API is a service that should be run by Quoters and interfaces that allow Solvers to post quotes.  The RFQ API takes in a yaml config that allows the user to specify which contracts, chains and interfaces it should run on. The config is structured like this:

```yaml
database:
  type: mysql # can be other mysql or sqlite
  dsn: root:password@hostname:3306)/database?parseTime=true # should be the dsn of your database. If using sqlite, this can be a path
omnirpc_url: https://route-to-my-omnirpc # omnirpc route
bridges:
  1: '0x00......' # FastBridge address on ethereum (chain id: 1)
  10: '0x01....' # FastBridge address on op (chain id: 10)
port: '8081'  # port to run your http server on
```

Yaml settings:

 - `database` - The database settings for the API backend. A database is required to store quotes and other information. Using SQLite with a dsn set to a `/tmp/` directory is recommended for development.
   -  `type` - the database driver to use, can be `mysql` or `sqlite`.
   -  `dsn` - the dsn of your database. If using sqlite, this can be a path, if using mysql please see [here](https://dev.mysql.com/doc/connector-odbc/en/connector-odbc-configuration.html) for more information.
 - `omnirpc_url` - The omnirpc url to use for querying chain data (no trailing slash). For more information on omnirpc, see [here](/docs/Services/Omnirpc).
 - `bridges` - A key value map of chain id to FastBridge contract address. The API will only allow quotes to be posted on these chains.
 - `port` - The port to run the http server on.

**Building From Source:**

To build the RFQ API from source, you will need to clone the repository and run the main.go file with the config file. Building from source requires go 1.21 or higher and is generally not recommended for end-users.

1. `git clone https://github.com/synapsecns/sanguine --recursive`
2. `cd sanguine/services/rfq`
3. `go run main.go --config /path/to/config.yaml`



**Running with Docker**

The RFQ API can also be run with docker. To do this, you will need to build the docker image and run it with the config file.

:::tip
Docker versions should always be pinned in production enviornments. For a full list of tags, see [here](https://github.com/synapsecns/sanguine/pkgs/container/sanguine%2Frfq-api)
:::

1. `docker run ghcr.io/synapsecns/sanguine/rfq-api:latest --config /path/to/config`
