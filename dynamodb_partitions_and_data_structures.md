# DynamoDB Partitions and Internal Data Structures

## How DynamoDB Partitions Work

### Partition Key Hashing
- DynamoDB uses a hash function on the partition key to determine which partition stores an item
- Hash function distributes data evenly across available partitions
- Each partition can store up to 10GB of data and handle 3,000 RCU or 1,000 WCU

### Automatic Partitioning
DynamoDB automatically creates new partitions when:
- Storage exceeds 10GB per partition
- Throughput exceeds partition limits
- Hot partitions are detected

### Partition Distribution Flow
```
Item → Hash(Partition Key) → Partition Assignment → Physical Storage
```

## Internal Data Structures

### B-Trees vs B+ Trees

**B-Trees:**
- Data stored in both internal and leaf nodes
- Each node contains keys and data
- Search may end at any level
- Less efficient for range scans

**B+ Trees:**
- Data only stored in leaf nodes
- Internal nodes only contain keys for routing
- All leaf nodes linked together (linked list)
- Better for range queries and sequential access

```
B-Tree:           B+ Tree:
   [K1|D1]           [K1|K2]
   /     \           /      \
[K2|D2] [K3|D3]   [D1] -> [D2] -> [D3]
```

### Primary Data Structure: B+ Trees
- Each partition uses B+ trees for efficient range queries
- Leaf nodes contain actual data items
- Internal nodes contain routing information
- Optimized for both point lookups and range scans

### Hash Tables for Partition Routing
- Global hash table maps partition key ranges to physical partitions
- Local hash tables within partitions for fast item lookup
- Consistent hashing for partition rebalancing

### LSM Trees (Log-Structured Merge Trees)
- Used for write-heavy workloads
- Writes go to in-memory structures first
- Periodic compaction merges data to disk
- Enables high write throughput

## Read and Write Paths

### Read Path
1. Hash(partition_key) → finds partition number
2. Hash tables → locate partition
3. B+ tree within partition → finds exact item location
4. Return data to client

### Write Path with LSM Trees

**1. Memory First (MemTable):**
```
Write Request → MemTable (in-memory sorted structure)
                    ↓
              WAL (Write-Ahead Log for durability)
```

**2. Background Compaction:**
```
MemTable (full) → SSTable (disk segment)
                      ↓
              Periodic compaction merges SSTables
```

**3. Read Path Complexity:**
```
Read Request → Check MemTable first
                ↓ (if not found)
              Check SSTables (newest to oldest)
                ↓
              Merge results and return
```

## Why LSM Trees for Writes?

### Sequential Writes are Faster
- Writes go to memory first (fast)
- Periodic flush to disk in large batches
- No random disk I/O during writes

### Write Amplification Trade-off
- Writes are amplified during compaction
- But individual writes are much faster
- Good for write-heavy workloads

## DynamoDB's Hybrid Approach

DynamoDB likely uses:
- **LSM trees** for the write path (fast ingestion)
- **B+ trees** for the final storage structure (efficient reads)
- **Compaction process** converts LSM segments to B+ tree format

### Simplified Flow
```
Write → MemTable → SSTable segments → Compaction → B+ Tree storage
Read → Hash table → B+ Tree → Item location
```

## Indexing Structures

### GSI/LSI (Global/Local Secondary Indexes)
- Separate B+ trees with different key structures
- Sparse indexes: Only items with index attributes are stored
- Projection: Controls which attributes are copied to indexes

## Key Design Principles

### Hot Partition Avoidance
- Use high-cardinality partition keys
- Avoid sequential patterns (timestamps, auto-incrementing IDs)
- Consider composite keys for better distribution

### Access Patterns
- Partition key determines physical location
- Sort key enables range queries within partition
- Cross-partition queries require scans (expensive)

### Example of Good vs Bad Partition Keys
```
❌ Bad: timestamp (creates hot partitions)
✅ Good: user_id + timestamp (distributes load)
```

## Summary

The combination of hash-based partitioning with B+ trees provides DynamoDB's ability to scale horizontally while maintaining single-digit millisecond latency. The hybrid approach using LSM trees for writes and B+ trees for reads gives DynamoDB both fast writes and fast reads, enabling it to handle both read and write-heavy workloads efficiently.
