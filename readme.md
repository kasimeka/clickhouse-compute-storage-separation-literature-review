<!-- markdownlint-disable line-length blanks-around-lists no-inline-html -->
<!-- cspell: words clickhouse inodes lifecycles altinity NVME BYOC architectured Karpenter FOSS failovers roadmaps Alexey Milovidov swmr -->
# state of compute-storage separation in clickhouse

## objective

this document provides a _literature review_ of the current state of stateless servers (compute-storage separation) in clickhouse as of Jan 7th 2026. it evaluates the available architectures and their limitations

## the proprietary standard: SharedMergeTree, ClickHouseⓇ Cloud and ClickHouseⓇ Private

the closed source, ClickHouseⓇ Inc. exclusive `SharedMergeTree` engine represents the gold standard for CH compute-storage separation  
[![`SharedMergeTree` architecture overview](https://clickhouse.com/docs/assets/ideal-img/shared-merge-tree-2.3771d1a.1024.png)](https://clickhouse.com/docs/cloud/reference/shared-merge-tree)

- persistence is managed entirely within shared storage (S3) for data parts and Keeper as a centralized metadata store
- compute nodes are completely stateless and fully decoupled; no communication takes place between them, everything flows through Keepers and S3
- it provides lightweight, asynchronous, leaderless replication
  - writes sent to any node are stored in S3 and their metadata are replicated to a Keeper quorum
  - each replica independently fetches new metadata in the background and discovers the new parts
  - nothing needs to be replicated as the cluster scales out
  - inserts/mutations/merges are all faster with lower latency courtesy of operating directly on a single shared copy of the data

### SMT is only available through ClickHouseⓇ Inc.'s services

1. ClickHouseⓇ Cloud, default SaaS offering in [limited regions](https://clickhouse.com/docs/cloud/reference/supported-regions#aws-regions)
1. [ClickHouseⓇ Private](https://clickhouse.com/docs/cloud/infrastructure/clickhouse-private)
   - a fully self-hosted packaging of SMT engine code
   - architectured as a lightweight stateless API (HTTP & TCP), horizontally scalable cache layer(s) managed by a K8s operator backed by a shared S3 bucket. no mention of how/where Keeper is hosted but it's likely managed by the operator too.
     - we can likely autoscale it with KEDA, optimize its compute with Karpenter, trace it with eBPF etc etc
   - license costs $0.5M a year
   - not ready for production use, requires a lot of manual setup and management by their support and has exactly two customers according to the sales call
     - although if it's actually a K8s operator it should be installable with helm and able to run on autopilot
   - the sales rep said only documented features are production ready -> there aren't really any documented features yet :D
   - in conclusion, it doesn't exist yet and they're not really selling it
1. [ClickHouseⓇ Cloud BYOC](https://clickhouse.com/docs/cloud/reference/byoc/overview)
   - limited to the ClickHouseⓇ Cloud regions
   - a limited subset of CH Cloud features, most notably missing autoscaling
   - we provide CH (almost) full access to an AWS account, VPC and S3 bucket
   - they deploy a CH Cloud data plane (on an EKS cluster), prometheus stack, tailscale, etc. only their engineers can access this environment
   - they connect it to the central CH Cloud control plane for billing and management

in conclusion `SharedMergeTree` is the correct architectural solution, but its distribution model and exclusivity make it non-viable for self-hosted environments

## the future candidate: altinity antalya

altinity’s project antalya aims to provide an open-source mechanism for CH disaggregated storage by integrating it with open data lake formats (Apache Iceberg)  
[![Antalya architecture overview](https://docs.altinity.com/images/altinitycloud/Antalya-Reference-Architecture-2025-02-17.png)](https://docs.altinity.com/altinityantalya/antalya-concepts/#overview)

while antalya is promising, it's currently in a very early stage of development, not recommended for production use, and and only implements Iceberg support and stateless reader swarms; the writer path (writes, mutations & merges) remains stateful and much more complex than default *MergeTree tables;

according to the Project Antalya 2025 Roadmap's [progress as of Jan 2026](https://web.archive.org/web/20260109153321/https://github.com/Altinity/ClickHouse/issues/804), the most notable antalya limitations are as follows:
- clients can't write directly into Iceberg, you write to a stateful server with *MergeTree tables (representing hot data)
- you're expected to run background jobs (using cron or their `ice` CLI) to `ALTER TABLE ... EXPORT PART` local *MergeTree parts to remote storage, these exported parts aren't automatically dropped from stateful servers, you need to drop them yourself as well
- you create `ENGINE=Hybrid` tables that can query both local and remote parts based on a "time watermark", which is a specific timestamp you partition your queries by where data older than the watermark is read from Iceberg and newer data is read from servers

## the ClickHouse FOSS status quo: ReplicatedMergeTree on S3

for self-hosted environments, the standard method for using object storage is the `ReplicatedMergeTree` engine configured with an S3 storage policy. this architecture allows data to reside on S3 which is cheaper and infinitely scalable, but fails to achieve compute-storage separation due to

- storage of table metadata on server disks
- severe limitations of the replication mechanism

### the standard operation of RMT on S3

since CH 22.8, by default, writes to S3-backed RMT tables happen as follows:
1. node A writes a part to S3
2. node A informs the cluster via Keeper that a part was written
3. node B (a replica) sees the "new part" metadata
4. node B proceeds to download its data from node A over the network
5. node B processes the data and uploads a new copy to a different path in the bucket
6. node B acknowledges the replication in Keeper

![RMT on S3 without zero-copy replication](https://altinity.com/wp-content/uploads/2024/05/MergeTree-S3-04-part-replication.png)

this process
- results in `n` copies of the data in S3 for a cluster with `n` replicas
- saturates network and disk bandwidth with redundant data transfer
- doesn't provide any of the cost or efficiency benefits of compute-storage separation

### the original plan for RMT on S3 (zero-copy replication)

to avoid the previously mentioned redundancies, zero-copy replication (`allow_remote_fs_zero_copy_replication`) was introduced alongside remote storage backends and was initially on by default.

![RMT on S3 with zero-copy replication](https://altinity.com/wp-content/uploads/2024/05/MergeTree-S3-05-zero-copy.png)

- it allows a replica to simply update its local metadata to point to S3 objects written by other replicas, instead of downloading and re-uploading data
- the initial implementation was unsound leading to data corruption and undefined consistency semantics.
  - the feature was turned off by default in v22.8 then re-marked as experimental in v25.8
  - its implementation was considered incomplete and unsafe,
  - and since ~april 2024 all contributions to its code are rejected with the message "we don't support zero-copy replication"

## the CH FOSS cutting edge: non-replicated MergeTree on s3_plain_rewritable storage

<https://clickhouse.com/docs/operations/storing-data#s3-plain-rewritable-storage>  
<https://github.com/ClickHouse/ClickHouse/pull/79047>  

`s3_plain_rewritable` disks store table data on s3 using standard, readable filenames instead of `s3`'s opaque blob ids and pointers, they also store all metadata directly in S3 rather than on the local filesystem, making server nodes _theoretically_ stateless

> [!NOTE]
> since metadata is stored in S3 as 1~10 files per table, this setup isn’t cost efficient to scale the table count _infinitely_ because the cost of polling table metadata will add up significantly; for this use case only `SharedMergeTree` is viable. `s3_plain_rewritable` only good for scaling a moderate number of tables to huge sizes, not the other way around.

### limitations

- Keeper coordination is not supported for this storage backend, so
  - it can't be used with `ReplicatedMergeTree` engine tables
  - in its current state, multiple writers can attach the same table simultaneously without coordination, leading to inconsistencies (writes by a specific node aren't automatically visible to other writers, without a restart) and -imo, not sure if proven- possible data corruption
  - as of v25.4, readonly MergeTree tables can be configured to poll their storage for metadata changes, allowing multiple readers to share the same `s3_plain_rewritable` disk and attach its tables
- single writer architecture in turn imposes the following limitations:
  1. no high availability, fault tolerance or _atomic_ failovers (old writer has to be fully shut down before a new one can attach tables),  tho Alexey created <https://github.com/ClickHouse/ClickHouse/issues/91613> to add support for stand-by writers that allow for zero-downtime failovers
  1. storage tiering, which would be of interest to prevent executing merges in S3, is a risk since only a single copy of hot data will be present on one node, and losing it will lose all hot data
- mutations (`ALTER TABLE ... DELETE/UPDATE` queries) aren't supported, and all table migrations require creating new tables and copying data over to them, due to S3's lack of support for hard links and atomic renames, emulating these functionalities for `plain_rewritable` storage was added to the ClickHouse 2026 roadmap <https://github.com/ClickHouse/ClickHouse/issues/91611>

### architecture

the feature set and limitations of `s3_plain_rewritable` storage imposes a single specific architecture:
- one (and only one) writer node owns and manages the table lifecycle
- multiple readonly reader nodes poll tables in S3 for metadata changes and attach/detach parts as needed
- all nodes are stateless and can be scaled independently with no overhead
  - writer statelessness is not documented/advertised so it's still _pending my investigation_

<p align="center"><img
  alt="s3_plain_rewritable single-writer multi-reader architecture diagram"
  src="./clickhouse-swmr.drawio.svg"
  style="max-width: 50%"
/></p>

this single-writer multi-reader architecture is not documented on clickhouse.com nor is it supported by / included on any ClickHouseⓇ Inc. roadmaps, it's pretty much a pet project of Alexey Milovidov
