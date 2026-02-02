### [Replication in DocDB](https://docs.yugabyte.com/stable/architecture/docdb-replication/replication/)
[Replication factor](https://docs.yugabyte.com/stable/architecture/docdb-replication/replication/#replication-factor)
YugabyteDB replicates data across fault domains (which, depending on the deployment, could be nodes, availability zones, or regions) in order to tolerate faults.  
The replication factor **(RF) is the number of copies of data** in a YugabyteDB cluster.  

Each tablet comprises of **a set of tablet peers**, each of which stores one copy of the data belonging to the tablet.  
There are as many tablet peers for a tablet **as the replication factor**, and they form a Raft group.  
The tablet peers are **hosted on different nodes** to allow data redundancy to protect against node failures.  
**The replication** of data between the tablet peers **is strongly consistent**.
