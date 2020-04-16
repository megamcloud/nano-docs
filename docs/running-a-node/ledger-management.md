# Ledger Management

The node automatically manages the full Nano ledger in the `data.ldb` file which can be found in the data folder at these locations:

--8<-- "folder-locations.md"

This file will grow in size as the ledger does. As of April 2020 there are over 49 million blocks in the ledger which requires at least 26GB of free space. See [hardware recommendations](/running-a-node/node-setup/#hardware-recommendations) for more preferred node specs.

!!! warning "RocksDB uses many files"
	The above details are for the default LMDB database setup. If using RocksDB, please note that it uses potentially 100s of SST files to manage the ledger so details should followed from the [RocksDB Ledger Backend](#rocksDB-ledger-backend) section below.

## Bootstrapping

When starting a new node the ledger must be downloaded and kept updated in order to participate on the network properly. This is done automatically via bootstrapping - the node downloads and verifies blocks from other nodes across the network. This process can take hours to days to complete depending on network conditions and [hardware specifications](/running-a-node/node-setup/#hardware-recommendations).

### Tuning options

**TODO**: Consider adding notes about tuning here and link to related config entries that need to be updated on configuration page: `bootstrap_connections_max` and `bootstrap_initiator_threads`? Maybe: "Depending on machine and networking resources, the bootstrap performance can be improved by updating X and X configuration values. The additional resource usage these options cause should be considered, especially if left during normal operation (after initial bootstrap is complete)."

## Ledger Fast Sync

!!! tip "Always backup your ledgers file"
	Whenever you are attempting to change the ledger, it is highly recommended you create backups of the existing `data.ldb` file to ensure you have a rollback point if issues are encountered.

To avoid bootstrapping times, a ledger file (`data.ldb`) can be downloaded off-chain and added to the data file used by the node. The Nano Foundation provides a daily ledger file download in the #ledger channel of our [Discord server](https://chat.nano.org). This is posted by `SergSW` and contains checksums for validation.

Before using this method there are a few considerations to ensure it is done safely:

### Data source
Make sure you trust the source providing the data to you. If you are unfamiliar with the individual or organization providing the ledger, consider other options for the data or fallback to the default of [bootstrapping](#bootstrapping) from the network.

### Voting weights
Blocks are confirmed using the voting weight of representatives and these weights are determined by the account balances assigned to those representatives. In addition, the node releases contain a hard-coded set of representative weights captured at the time of release.

If a new ledger is downloaded it is recommended to compare the voting weights it contains against the hard-coded values in the node. To do this, the [--compare_rep_weights](/commands/command-line-interface/#-compare_rep_weights) CLI (_available in V21+ only_) can be run to get an output of the variance.

Although the variance threshold considered to be safe will depend on freshness of the ledger file and network weight variations since the node release date, in general more than a 10% difference should be scrutinized further. If you need support in evaluating join the [Node and Representative Management category](https://forum.nano.org/c/node-and-rep/8) on the [Nano Forums](https://forum.nano.org).

### Confirmation data
Within each account on the ledger a confirmation height is set. This indicates the height of the last block on that chain where quorum was observed on the network. This is set locally by the node and a new ledger file may include this information with it. If the ledger is from a trusted source this confirmation data can be kept, which will save bandwidth and resources on the network by not querying for votes to verify these confirmations.

If confirmation data for the ledger is not trusted the [--confirmation_height_clear](/commands/command-line-interface/#-confirmation_height_clear) CLI can be used to clear these out.

---

## RocksDB Ledger Backend

!!! warning "RocksDB is experimental, do not use in production"
	RocksDB is being included in _V20.0_ as experimental only. Future versions of the node may allow for production use of RocksDB, however old experimental RocksDB ledgers are not guarenteed to be compatible and may require resyncing from scratch.

	If you are testing RocksDB and want to discuss results, configurations, etc. please join the forum topic here: https://forum.nano.org/t/rocksdb-ledger-backend-testing/111

The node ledger currently uses LMDB (Lightning memory-mapped database) by default as the data store. As of _v20+_ the option to use RocksDB becomes available as an experimental option.
This document will not go into much detail about theses key-value data stores as there is a lot of information available online.
It is anticipated that bootstrapping will be slower using RocksDB during the initial version at least, but live traffic should be faster due to singluar writes being cached in memory and flushed to disk in bulk.

Using RocksDB requires a few extra steps as it is an externally required dependency which requires a recent version of RocksDB, so older repositories may not be sufficient, it also requires `zlib`. If using the docker node, can skip to [Enable RocksDB](#enable-rocksdb):  

### Installation  

**Linux**  
Ubuntu 19.04 and later:
```
sudo apt-get install zlib1g-dev
sudo apt-get install librocksdb-dev
```
Otherwise:
```
sudo apt-get install zlib1g-dev
export USE_RTTI=1
git clone https://github.com/facebook/rocksdb.git
cd rocksdb
make static_lib
make install
```
**MacOS**  
`brew install rocksdb`

**Windows**  
Recommended way is to use `vcpkg`:

* add `set (VCPKG_LIBRARY_LINKAGE static)` to the top of `%VCPKG_DIR%\ports\rocksdb\portfile.cmake`
* `vcpkg install rocksdb:x64-windows`

For other or more detailed instructions visit the official page:
https://github.com/facebook/rocksdb/blob/master/INSTALL.md

### Build node with RocksDB support
Once RocksDB is installed successfully, the node must be built with RocksDB support using the CMake variable `-DNANO_ROCKSDB=ON`

The following CMake options can be used to specify where the RocksDB and zlib libraries are if they cannot be found automatically:
```
ROCKSDB_INCLUDE_DIRS
ROCKSDB_LIBRARIES
ZLIB_LIBRARY
ZLIB_INCLUDE_DIR
```
### Enable RocksDB
This can be enabled by adding the following to the `config-node.toml` file:

```
[node.rocksdb]
enable = true
```

There are many other options which can be set. Due to RocksDB generally using more memory the defaults have been made pessimistic in order to run on a wider range of lower end devices. Recommended settings if on a system with 8GB or more RAM (see TOML comments in the generated file for more information on what these do):

```
[node.rocksdb]
bloom_filter_bits = 10
block_cache = 1024
enable_pipelined_write=true
cache_index_and_filter_blocks=true
block_size=64
memtable_size=128
num_memtables=3
total_memtable_size=0
```
Comparision:

| LMDB | RocksDB |
| :-------: | :---------: |
| Tested with the node for many years | Experimental status |
| 1 file (data.ldb) | 100+ SST files |
| *15GB live ledger size | Smaller file size (11GB) |
| Not many options to configure  | Very configurable |
| Unlikely to be further optimized | Many optimizations possible in future |
| Part of the node build process | Required external dep (incl recent version 5.13+) |
| - | Less file I/O (writes are flushed in bulk) |
| - | May use more memory |

\* At the time of writing (Oct 2019)

RocksDB Limitations:

* Automatic backups not currently supported
* Database transaction tracker is not supported
* Cannot execute CLI commands which require writing to the database, such as `nano_node --peer_clear` these must be executed when the node is stopped

!!! note "Snapshotting with RocksDB"
	When backing up using the --snapshot CLI option, it is currently set up to do incremental backups, which reduces the need to copy the whole database. However if the original files are deleted, then the backup directory should also be deleted otherwise there can be inconsistencies.