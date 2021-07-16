<p align="center">
  <a href="https://www.rosetta-api.org">
    <img width="90%" alt="Rosetta" src="https://www.rosetta-api.org/img/rosetta_header.png">
  </a>
</p>
<h3 align="center">
   Rosetta UnelmaCoin
</h3>
<p align="center">
  <a href="https://circleci.com/gh/unelmacoin/rosetta-unelmacoin/tree/master"><img src="https://circleci.com/gh/unelmacoin/rosetta-unelmacoin/tree/master.svg?style=shield" /></a>
  <a href="https://coveralls.io/github/unelmacoin/rosetta-unelmacoin"><img src="https://coveralls.io/repos/github/unelmacoin/rosetta-unelmacoin/badge.svg" /></a>
  <a href="https://goreportcard.com/report/github.com/unelmacoin/rosetta-unelmacoin"><img src="https://goreportcard.com/badge/github.com/unelmacoin/rosetta-unelmacoin" /></a>
  <a href="https://github.com/unelmacoin/rosetta-unelmacoin/blob/master/LICENSE.txt"><img src="https://img.shields.io/github/license/unelmacoin/rosetta-unelmacoin.svg" /></a>
  <a href="https://pkg.go.dev/github.com/unelmacoin/rosetta-unelmacoin?tab=overview"><img src="https://img.shields.io/badge/go.dev-reference-007d9c?logo=go&logoColor=white&style=shield" /></a>
</p>

<p align="center"><b>
ROSETTA-UNELMACOIN IS CONSIDERED <a href="https://en.wikipedia.org/wiki/Software_release_life_cycle#Alpha">ALPHA SOFTWARE</a>.
USE AT YOUR OWN RISK! COINBASE ASSUMES NO RESPONSIBILITY NOR LIABILITY IF THERE IS A BUG IN THIS IMPLEMENTATION.
</b></p>

## Overview
`rosetta-unelmacoin` provides a reference implementation of the Rosetta API for
Bitcoin in Golang. If you haven't heard of the Rosetta API, you can find more
information [here](https://rosetta-api.org).

## Features
* Rosetta API implementation (both Data API and Construction API)
* UTXO cache for all accounts (accessible using `/account/balance`)
* Stateless, offline, curve-based transaction construction from any SegWit-Bech32 Address

## Usage
As specified in the [Rosetta API Principles](https://www.rosetta-api.org/docs/automated_deployment.html),
all Rosetta implementations must be deployable via Docker and support running via either an
[`online` or `offline` mode](https://www.rosetta-api.org/docs/node_deployment.html#multiple-modes).

**YOU MUST INSTALL DOCKER FOR THE FOLLOWING INSTRUCTIONS TO WORK. YOU CAN DOWNLOAD
DOCKER [HERE](https://www.docker.com/get-started).**

### Install
Running the following commands will create a Docker image called `rosetta-unelmacoin:latest`.

#### From GitHub
To download the pre-built Docker image from the latest release, run:
```text
curl -sSfL https://raw.githubusercontent.com/unelmacoin/rosetta-unelmacoin/master/install.sh | sh -s
```
_Do not try to install rosetta-unelmacoin using GitHub Packages!_

#### From Source
After cloning this repository, run:
```text
make build-local
```

### Run
Running the following commands will start a Docker container in
[detached mode](https://docs.docker.com/engine/reference/run/#detached--d) with
a data directory at `<working directory>/bitcoin-data` and the Rosetta API accessible
at port `8080`.

#### Mainnet:Online
```text
docker run -d --rm --ulimit "nofile=100000:100000" -v "$(pwd)/bitcoin-data:/data" -e "MODE=ONLINE" -e "NETWORK=MAINNET" -e "PORT=8080" -p 8080:8080 -p 8333:8333 rosetta-unelmacoin:latest
```
_If you cloned the repository, you can run `make run-mainnet-online`._

#### Mainnet:Offline
```text
docker run -d --rm -e "MODE=OFFLINE" -e "NETWORK=MAINNET" -e "PORT=8081" -p 8081:8081 rosetta-unelmacoin:latest
```
_If you cloned the repository, you can run `make run-mainnet-offline`._

#### Testnet:Online
```text
docker run -d --rm --ulimit "nofile=100000:100000" -v "$(pwd)/bitcoin-data:/data" -e "MODE=ONLINE" -e "NETWORK=TESTNET" -e "PORT=8080" -p 8080:8080 -p 18333:18333 rosetta-unelmacoin:latest
```
_If you cloned the repository, you can run `make run-testnet-online`._

#### Testnet:Offline
```text
docker run -d --rm -e "MODE=OFFLINE" -e "NETWORK=TESTNET" -e "PORT=8081" -p 8081:8081 rosetta-unelmacoin:latest
```
_If you cloned the repository, you can run `make run-testnet-offline`._

## System Requirements
`rosetta-unelmacoin` has been tested on an [AWS c5.2xlarge instance](https://aws.amazon.com/ec2/instance-types/c5).
This instance type has 8 vCPU and 16 GB of RAM.

### Network Settings
To increase the load `rosetta-unelmacoin` can handle, it is recommended to tune your OS
settings to allow for more connections. On a linux-based OS, you can run the following
commands ([source](http://www.tweaked.io/guide/kernel)):
```text
sysctl -w net.ipv4.tcp_tw_reuse=1
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216
sysctl -w net.ipv4.tcp_max_syn_backlog=10000
sysctl -w net.core.somaxconn=10000
sysctl -p (when done)
```
_We have not tested `rosetta-unelmacoin` with `net.ipv4.tcp_tw_recycle` and do not recommend
enabling it._

You should also modify your open file settings to `100000`. This can be done on a linux-based OS
with the command: `ulimit -n 100000`.

### Memory-Mapped Files
`rosetta-unelmacoin` uses [memory-mapped files](https://en.wikipedia.org/wiki/Memory-mapped_file) to
persist data in the `indexer`. As a result, you **must** run `rosetta-unelmacoin` on a 64-bit
architecture (the virtual address space easily exceeds 100s of GBs).

If you receive a kernel OOM, you may need to increase the allocated size of swap space
on your OS. There is a great tutorial for how to do this on Linux [here](https://linuxize.com/post/create-a-linux-swap-file/).

## Architecture
`rosetta-unelmacoin` uses the `syncer`, `storage`, `parser`, and `server` package
from [`rosetta-sdk-go`](https://github.com/coinbase/rosetta-sdk-go) instead
of a new Bitcoin-specific implementation of packages of similar functionality. Below
you can find a high-level overview of how everything fits together:
```text
                               +------------------------------------------------------------------+
                               |                                                                  |
                               |                 +--------------------------------------+         |
                               |                 |                                      |         |
                               |                 |                 indexer              |         |
                               |                 |                                      |         |
                               |                 | +--------+                           |         |
                               +-------------------+ pruner <----------+                |         |
                               |                 | +--------+          |                |         |
                       +-------v----+            |                     |                |         |
                       |unelmacoind |            |              +------+--------+       |         |
                       +-------+----+            |     +--------> block_storage <----+  |         |
                               |                 |     |        +---------------+    |  |         |
                               |                 | +---+----+                        |  |         |
                               +-------------------> syncer |                        |  |         |
                                                 | +---+----+                        |  |         |
                                                 |     |        +--------------+     |  |         |
                                                 |     +--------> coin_storage |     |  |         |
                                                 |              +------^-------+     |  |         |
                                                 |                     |             |  |         |
                                                 +--------------------------------------+         |
                                                                       |             |            |
+-------------------------------------------------------------------------------------------+     |
|                                                                      |             |      |     |
|         +------------------------------------------------------------+             |      |     |
|         |                                                                          |      |     |
|         |                     +---------------------+-----------------------+------+      |     |
|         |                     |                     |                       |             |     |
| +-------+---------+   +-------+---------+   +-------+-------+   +-----------+----------+  |     |
| | account_service |   | network_service |   | block_service |   | construction_service +--------+
| +-----------------+   +-----------------+   +---------------+   +----------------------+  |
|                                                                                           |
|                                         server                                            |
|                                                                                           |
+-------------------------------------------------------------------------------------------+
```

### Optimizations
* Automatically prune bitcoind while indexing blocks
* Reduce sync time with concurrent block indexing
* Use [Zstandard compression](https://github.com/facebook/zstd) to reduce the size of data stored on disk
without needing to write a manual byte-level encoding

#### Concurrent Block Syncing
To speed up indexing, `rosetta-unelmacoin` uses concurrent block processing
with a "wait free" design (using channels instead of sleeps to signal
which threads are unblocked). This allows `rosetta-unelmacoin` to fetch
multiple inputs from disk while it waits for inputs that appeared
in recently processed blocks to save to disk.
```text
                                                   +-------------+
                                                   | unelmacoind |
                                                   +-------+-----+
                                                         |
                                                         |
          +---------+ fetch block data / unpopulated txs |
          | block 1 <------------------------------------+
          +---------+                                    |
       +-->   tx 1  |                                    |
       |  +---------+                                    |
       |  |   tx 2  |                                    |
       |  +----+----+                                    |
       |       |                                         |
       |       |           +---------+                   |
       |       |           | block 2 <-------------------+
       |       |           +---------+                   |
       |       +----------->   tx 3  +--+                |
       |                   +---------+  |                |
       +------------------->   tx 4  |  |                |
       |                   +---------+  |                |
       |                                |                |
       | retrieve previously synced     |   +---------+  |
       | inputs needed for future       |   | block 3 <--+
       | blocks while waiting for       |   +---------+
       | populated blocks to save to    +--->   tx 5  |
       | disk                               +---------+
       +------------------------------------>   tx 6  |
       |                                    +---------+
       |
       |
+------+--------+
|  coin_storage |
+---------------+
```

## Testing with rosetta-cli
To validate `rosetta-unelmacoin`, [install `rosetta-cli`](https://github.com/coinbase/rosetta-cli#install)
and run one of the following commands:
* `rosetta-cli check:data --configuration-file rosetta-cli-conf/testnet/config.json`
* `rosetta-cli check:construction --configuration-file rosetta-cli-conf/testnet/config.json`
* `rosetta-cli check:data --configuration-file rosetta-cli-conf/mainnet/config.json`

## Future Work
* Publish benchamrks for sync speed, storage usage, and load testing
* [Rosetta API `/mempool/transaction`](https://www.rosetta-api.org/docs/MempoolApi.html#mempooltransaction) implementation
* Add CI test using `rosetta-cli` to run on each PR (likely on a regtest network)
* Add performance mode to use unlimited RAM (implementation currently optimized to use <= 16 GB of RAM)
* Support Multi-Sig Sends

_Please reach out on our [community](https://community.rosetta-api.org) if you want to tackle anything on this list!_

## Development
* `make deps` to install dependencies
* `make test` to run tests
* `make lint` to lint the source code
* `make salus` to check for security concerns
* `make build-local` to build a Docker image from the local context
* `make coverage-local` to generate a coverage report

## License
This project is available open source under the terms of the [Apache 2.0 License](https://opensource.org/licenses/Apache-2.0).

© 2020 Coinbase
