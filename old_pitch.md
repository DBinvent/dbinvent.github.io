# DBinvent

**_Build your own data source, better than on cloud_**

*Chicago based deep tech startup since 2019*

---

https://youtu.be/Ja-EIV2dzF0

_The concept of a problem we solve as simple as a HDD disk swap like RAID or NAS / Network storage, but for a database._
_Existing solutions works but (a) it's costy and/or (b) cloud/vendor lock & required a data transfer/downtime or non-reliable in matter of data consistency._
_The database is an old technology, but almost all internet websites use it, and when you needs a reliability with some complex solutions, plus performance, than you needs something new._


## Problems, that many of IT projects faced

### Data consistency - when transactions are not about money transfer
AWS: "The second Enactor's clean-up process then deleted this older plan because it was many generations older than the plan it had just applied" - this is a transaction isolation issue which lead to global disaster.

### Performance on scale-up - when vertical is not an option
CAP theorem - you can't have either: Consistency, Availability or Partition Tolerance.

### Backup, restore and repeat - without compromise
That is possible, but will require a single entry point and additional manual action on failure, and double spending for unused resources.

### Complex solution exists, but the simple is possible
No solutions are as simple as RAID or NAS for Database which is a heart of many IT projects from just a website to distributed commuting.

---

## YaXaHa - The cluster like in cloud, but better and yours

[https://www.dbinvent.com/cluster/](https://www.dbinvent.com/cluster/) — ready to use **MVP**

### Strong consistency on write and eventual consistency on read
Split the performance to a 80% read and 20% write load. Always able to read, but no write until the data become actual.

### Virtual dynamic partitioning - the CAP theorem SSC
Dynamically defining transaction boundaries and spread writing load on different nodes if they are disjoint (patent provisioning) **with AI**.

### RAFT consensus for a Transaction Coordinator - not a single entry
The special node called TC act as arbiter to assign workers nodes. The arbiter does not need to process data.

### Flexible configuration
Programmatically defined synchronization rules allows splitting data to redundant for synchronization and ephemeral for a local use.

---

## Roadmap

### Serverless platform - almost done
Build your own platform like AWS lambda or Azure Server Function without a vendor lock.

### Web UI - in progress
Simplify cluster setup by using web UI and provide user-friendly cluster management, including schema changes (like Flyway) and testing.

### Testing - in todo as the most priority to prove the durability with a time
Perform comprehensive consistency, stress and load testing. Determine SLA and TPS.

### Sharding and Partitioning - the architecture and analysis phase
From MPP to Global hierarchical cluster model to substitute client-server interaction and even a MMO gaming / mobile API 
includes all configurations from a Consistent storage to Realtime processing.

****
- MPP - massive parallel (query) processing, which is a distributed database architecture that allows for the parallel processing of queries across multiple nodes.
 This architecture is designed to handle large volumes of data and complex queries by distributing the workload across multiple servers, 
enabling faster query execution and improved performance.
- Global hierarchical cluster model - a distributed database architecture that organizes nodes in a hierarchical structure,
 allowing for efficient data distribution and query processing across multiple levels of the hierarchy. 
 With a programmatically defined hierarchy, data can be partitioned and replicated across different levels,
 This model can improve scalability, fault tolerance, and performance by optimizing data placement and query routing based on the hierarchical relationships between nodes.
- MMO gaming backend framework - a backend architecture designed to support the massive scale and real-time interactions of multiplayer online games. 
 It typically involves distributed servers, load balancing, and data synchronization mechanisms to handle large numbers of concurrent players and ensure a seamless gaming experience.
- Consistent storage - a storage system that guarantees data consistency across multiple nodes or replicas, ensuring that all copies of the data remain synchronized and up-to-date. 
 This is crucial for distributed databases and applications that require strong consistency guarantees to prevent data anomalies and maintain data integrity.
- Real-time processing - the ability to process and respond to data changes as they occur, ensuring that applications can make informed decisions based on the most current information.
- Mobile API - a set of application programming interfaces (APIs) specifically designed for mobile applications, but with a Postgres backend. 
 These APIs enable mobile apps to interact with the database, retrieve and manipulate data, and perform various operations while ensuring efficient communication and data synchronization between the mobile client and the backend server.
 

