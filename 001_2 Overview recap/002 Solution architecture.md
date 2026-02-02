### [Replication in DocDB](https://docs.yugabyte.com/stable/architecture/docdb-replication/replication/)
[Replication factor](https://docs.yugabyte.com/stable/architecture/docdb-replication/replication/#replication-factor)
YugabyteDB replicates data across fault domains (which, depending on the deployment, could be nodes, availability zones, or regions) in order to tolerate faults.  
The replication factor **(RF) is the number of copies of data** in a YugabyteDB cluster.  

Each tablet comprises of **a set of tablet peers**, each of which stores one copy of the data belonging to the tablet.  
> **There are as many tablet peers for a tablet as the replication factor**, and they form a **Raft group**.
 
The tablet peers are **hosted on different nodes** to allow data redundancy to protect against node failures.  
**The replication** of data between the tablet peers **is strongly consistent**.

**Tablet leader replicates** among the tablet peers using Raft **to achieve strong consistency**.

The **Raft log** is used to ensure that the database **state-machine of a tablet** is replicated amongst the tablet peers with strict ordering and correctness guarantees.

**After the Raft log is replicated** to a majority of tablet-peers and successfully persisted on the majority, the **write is applied** into the DocDB document storage layer and is subsequently available for reads.   
**After the write is persisted on disk** by the document storage layer, the **write entries can be purged** from the Raft log. 

The recovery point objective (**RPO**) for each of these tablets **is 0**, meaning no data is lost in the failover to another zone.  
The recovery time objective (**RTO**) **is 3 seconds**, which is the time window for completing the failover and becoming operational out of the new zones

YugabyteDB offers **reading from followers** with relaxed guarantees.
