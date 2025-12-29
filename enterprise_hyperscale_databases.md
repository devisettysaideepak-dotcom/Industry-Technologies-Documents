# Enterprise and Hyperscale Database Systems

## Google's Proprietary Databases

### 1. Google Spanner

**Type:** Globally distributed SQL database with ACID transactions

#### Architecture
```
Global Distribution:
Region 1 (US-East) ← → Region 2 (Europe) ← → Region 3 (Asia)
     ↓                      ↓                      ↓
Multiple Zones          Multiple Zones         Multiple Zones
     ↓                      ↓                      ↓
Paxos Consensus        Paxos Consensus       Paxos Consensus
```

#### Key Features
- **External Consistency**: Linearizable reads and writes globally
- **TrueTime API**: Uses GPS and atomic clocks for global ordering
- **Automatic Sharding**: Data automatically distributed across nodes
- **SQL Support**: Full SQL with joins, secondary indexes, transactions

#### Internal Architecture
**Storage Layer:**
- Data stored in tablets (horizontal partitions)
- Each tablet replicated across multiple zones
- Paxos consensus for consistency

**Transaction Layer:**
```
Transaction Coordinator
    ↓
Two-Phase Commit across regions
    ↓
Paxos Groups (per shard)
    ↓
Local Storage (Colossus file system)
```

#### Use Cases at Google
- **AdWords**: Financial transactions requiring global consistency
- **Google Play**: Purchase transactions across regions
- **Gmail**: User data with global access requirements

#### Performance Characteristics
- **Latency**: 10-100ms for cross-region transactions
- **Throughput**: Millions of transactions per second globally
- **Consistency**: Strong consistency with global ordering

---

### 2. Google Bigtable

**Type:** Wide-column NoSQL database for massive scale

#### Architecture
```
Client API
    ↓
Master Server (metadata)
    ↓
Tablet Servers (data serving)
    ↓
GFS/Colossus (distributed file system)
```

#### Data Model
```
Row Key: "com.google.www" (reversed domain)
Column Families: ["anchor", "contents", "metadata"]
Columns: anchor:cnnsi.com, anchor:my.look.ca
Timestamps: Multiple versions per cell
```

#### Storage Structure
**SSTable Format:**
```
Tablet (range of rows)
├── MemTable (in-memory, sorted)
├── SSTable 1 (immutable, on disk)
├── SSTable 2 (immutable, on disk)
└── SSTable N (compacted periodically)
```

#### Key Features
- **Petabyte Scale**: Handles petabytes of data across thousands of servers
- **High Throughput**: Millions of operations per second
- **Automatic Sharding**: Data automatically split and merged
- **Compression**: Efficient storage with compression algorithms

#### Use Cases at Google
- **Search Index**: Web crawl data and search indexes
- **Maps**: Geospatial data and imagery
- **YouTube**: Video metadata and user interactions
- **Analytics**: Click-stream data and user behavior

#### Performance Characteristics
- **Latency**: Single-digit milliseconds for reads
- **Throughput**: 100K+ operations per second per server
- **Scale**: Exabytes of data across global infrastructure

---

### 3. Google Firestore

**Type:** Document database with real-time synchronization

#### Architecture
```
Client SDKs (Web, Mobile, Server)
    ↓
Firestore API Gateway
    ↓
Document Storage + Real-time Engine
    ↓
Spanner (underlying storage)
```

#### Data Model
```javascript
// Collection → Document → Subcollection structure
users/{userId}/posts/{postId}/comments/{commentId}

// Document example
{
  name: "John Doe",
  email: "john@example.com",
  posts: [reference to posts subcollection],
  metadata: {
    created: timestamp,
    lastLogin: timestamp
  }
}
```

#### Key Features
- **Real-time Listeners**: Automatic UI updates on data changes
- **Offline Support**: Local caching with automatic sync
- **Multi-region**: Built on Spanner for global distribution
- **Security Rules**: Declarative security at database level

#### Use Cases at Google
- **Firebase Applications**: Mobile and web app backends
- **Real-time Collaboration**: Chat applications, collaborative editing
- **IoT Applications**: Device state synchronization

---

## Microsoft's Database Systems

### 1. Azure Cosmos DB

**Type:** Multi-model globally distributed database

#### Architecture
```
Global Distribution:
Primary Region ← → Secondary Regions (up to 30+)
    ↓
Consistency Levels (5 options)
    ↓
Multi-API Support (SQL, MongoDB, Cassandra, Gremlin, Table)
    ↓
Partitioned Storage Engine
```

#### Consistency Models
```
Strong ←→ Bounded Staleness ←→ Session ←→ Consistent Prefix ←→ Eventual
  ↑                                                                    ↑
Highest                                                            Lowest
Consistency                                                      Consistency
Lowest                                                            Highest
Performance                                                     Performance
```

#### Multi-Model Support
**Document API (SQL):**
```json
{
  "id": "user123",
  "name": "John Doe",
  "orders": [
    {"orderId": "order456", "total": 99.99}
  ]
}
```

**Graph API (Gremlin):**
```
g.V().hasLabel('person').has('name', 'John').out('knows').values('name')
```

**Key-Value API (Table):**
```
PartitionKey: "users"
RowKey: "user123"
Properties: {name: "John", email: "john@example.com"}
```

#### Key Features
- **Turnkey Global Distribution**: One-click multi-region deployment
- **Elastic Scale**: Automatic scaling based on demand
- **SLA Guarantees**: 99.999% availability, <10ms latency
- **Multiple APIs**: Use existing skills (MongoDB, Cassandra drivers)

#### Use Cases at Microsoft
- **Office 365**: User profiles and collaboration data
- **Xbox Live**: Gaming profiles and leaderboards
- **IoT Solutions**: Device telemetry and state management

---

### 2. SQL Server / Azure SQL Database

**Type:** Enterprise relational database with cloud capabilities

#### Architecture
```
Application Layer
    ↓
Query Processor (Optimizer, Execution Engine)
    ↓
Storage Engine (Buffer Pool, Lock Manager)
    ↓
Physical Storage (Data Files, Log Files, TempDB)
```

#### Advanced Features
**In-Memory OLTP:**
```sql
-- Memory-optimized tables
CREATE TABLE Users (
    UserID int IDENTITY(1,1) PRIMARY KEY NONCLUSTERED HASH WITH (BUCKET_COUNT = 1000000),
    Name nvarchar(100) NOT NULL
) WITH (MEMORY_OPTIMIZED = ON, DURABILITY = SCHEMA_AND_DATA);
```

**Columnstore Indexes:**
```sql
-- For analytics workloads
CREATE CLUSTERED COLUMNSTORE INDEX CCI_Sales ON Sales;
-- Compression ratio: 7-10x, query performance: 10-100x faster
```

**Always On Availability Groups:**
```
Primary Replica (Read/Write)
    ↓ (Synchronous/Asynchronous)
Secondary Replicas (Read-only)
    ↓
Automatic Failover + Load Balancing
```

#### Key Features
- **Enterprise Security**: Row-level security, encryption, auditing
- **High Availability**: 99.99% uptime with Always On
- **Intelligent Performance**: Automatic tuning and optimization
- **Hybrid Cloud**: Seamless on-premises to cloud migration

---

## Uber's Database Stack

### 1. MySQL (Heavily Customized)

**Uber's MySQL Architecture:**
```
Application Layer
    ↓
Connection Pooling (Vitess-like)
    ↓
Sharding Layer (Custom routing)
    ↓
MySQL Clusters (Master-Slave per shard)
    ↓
Storage (InnoDB with custom optimizations)
```

#### Sharding Strategy
```python
# User-based sharding
def get_shard(user_id):
    return user_id % NUM_SHARDS

# Geographic sharding
def get_shard(city_id):
    return CITY_TO_SHARD_MAPPING[city_id]

# Time-based sharding (for trips)
def get_shard(trip_timestamp):
    return trip_timestamp.year_month % NUM_SHARDS
```

#### Uber's MySQL Optimizations
- **Custom InnoDB**: Modified for better SSD performance
- **Connection Pooling**: Reduced connection overhead
- **Read Replicas**: Separate analytics workloads
- **Backup Strategy**: Point-in-time recovery across shards

#### Use Cases at Uber
- **User Profiles**: Driver and rider account information
- **Trip Data**: Trip history, ratings, payments
- **Financial Transactions**: Payment processing, driver earnings

---

### 2. Apache Cassandra

**Uber's Cassandra Architecture:**
```
Ring Topology (Consistent Hashing):
Node 1 ← → Node 2 ← → Node 3 ← → Node 4
  ↑                                    ↓
Node 8 ← ← ← ← ← ← ← ← ← ← ← ← ← ← Node 5
```

#### Data Modeling for Location Tracking
```cql
-- Time-series data for driver locations
CREATE TABLE driver_locations (
    driver_id UUID,
    time_bucket text,  -- "2024-01-01-14" (hour bucket)
    timestamp timestamp,
    latitude double,
    longitude double,
    speed double,
    heading int,
    PRIMARY KEY ((driver_id, time_bucket), timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC);
```

#### Geospatial Queries
```cql
-- Find drivers near location
CREATE TABLE drivers_by_geohash (
    geohash text,      -- "9q8yy" (precision 5)
    driver_id UUID,
    latitude double,
    longitude double,
    last_update timestamp,
    PRIMARY KEY (geohash, driver_id)
);
```

#### Key Features for Uber
- **High Write Throughput**: Millions of location updates per second
- **Tunable Consistency**: Balance between consistency and performance
- **Multi-DC Replication**: Global driver tracking
- **Time-Series Optimization**: Efficient storage of location history

#### Use Cases at Uber
- **Real-time Location Tracking**: Driver and rider positions
- **Trip Analytics**: Historical trip patterns
- **Surge Pricing Data**: Supply/demand metrics by location

---

### 3. Redis

**Uber's Redis Architecture:**
```
Application Servers
    ↓
Redis Cluster (Sharded)
├── Shard 1: Hash slots 0-5460
├── Shard 2: Hash slots 5461-10922  
└── Shard 3: Hash slots 10923-16383
    ↓
Persistence: RDB + AOF
```

#### Real-time Matching Algorithm
```python
# Driver availability in Redis
GEOADD drivers:available longitude latitude driver_id

# Find nearby drivers
GEORADIUS drivers:available user_longitude user_latitude 2 km WITHDIST

# Driver state management
HSET driver:123 status available lat 37.7749 lng -122.4194 last_update 1640995200

# Rate limiting
INCR rate_limit:user:456:minute
EXPIRE rate_limit:user:456:minute 60
```

#### Caching Strategy
```python
# Trip matching cache
SET trip_request:789 '{"user_id": 456, "pickup": {...}, "destination": {...}}' EX 300

# Session management  
HSET session:user:456 auth_token abc123 expires_at 1640995200

# Leaderboards (driver ratings)
ZADD driver_ratings 4.95 driver:123 4.87 driver:456
```

#### Use Cases at Uber
- **Real-time Matching**: Driver-rider pairing algorithms
- **Session Storage**: User authentication and state
- **Rate Limiting**: API throttling and abuse prevention
- **Caching**: Frequently accessed data (user profiles, pricing)

---

### 4. ClickHouse

**Uber's ClickHouse Architecture:**
```
Data Ingestion (Kafka) → ClickHouse Cluster
                              ↓
                    Columnar Storage (MergeTree)
                              ↓
                    Analytics Queries (SQL)
```

#### Schema Design for Analytics
```sql
-- Trip analytics table
CREATE TABLE trip_analytics (
    trip_id String,
    driver_id String,
    rider_id String,
    city_id UInt32,
    trip_start_time DateTime,
    trip_end_time DateTime,
    pickup_lat Float64,
    pickup_lng Float64,
    dropoff_lat Float64,
    dropoff_lng Float64,
    distance_km Float32,
    duration_minutes UInt16,
    fare_amount Decimal(10,2),
    surge_multiplier Float32,
    payment_method String,
    rating UInt8
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(trip_start_time)
ORDER BY (city_id, trip_start_time, trip_id);
```

#### Real-time Analytics Queries
```sql
-- Surge pricing calculation
SELECT 
    city_id,
    toStartOfHour(trip_start_time) as hour,
    count() as trip_count,
    avg(surge_multiplier) as avg_surge,
    percentile(0.95)(fare_amount) as p95_fare
FROM trip_analytics 
WHERE trip_start_time >= now() - INTERVAL 24 HOUR
GROUP BY city_id, hour
ORDER BY hour DESC;

-- Driver utilization metrics
SELECT 
    driver_id,
    count() as trips_completed,
    sum(duration_minutes) as total_driving_minutes,
    avg(rating) as avg_rating,
    sum(fare_amount) as total_earnings
FROM trip_analytics 
WHERE trip_start_time >= today()
GROUP BY driver_id;
```

#### Use Cases at Uber
- **Real-time Dashboards**: Business metrics and KPIs
- **Surge Pricing**: Supply/demand analytics
- **Driver Analytics**: Performance and earnings tracking
- **City Operations**: Traffic patterns and optimization

---

## Performance Comparison

### Latency Characteristics
| Database | Read Latency | Write Latency | Use Case |
|----------|--------------|---------------|----------|
| **Spanner** | 10-100ms | 50-200ms | Global transactions |
| **Bigtable** | 1-10ms | 1-10ms | Massive scale reads |
| **Cosmos DB** | 1-10ms | 1-10ms | Multi-region apps |
| **MySQL (Uber)** | 1-5ms | 1-5ms | Transactional data |
| **Cassandra** | 1-10ms | <1ms | Time-series data |
| **Redis** | <1ms | <1ms | Real-time operations |
| **ClickHouse** | 10-1000ms | 100-1000ms | Analytics queries |

### Scale Characteristics
| Database | Max Scale | Consistency | Availability |
|----------|-----------|-------------|--------------|
| **Spanner** | Global | Strong | 99.999% |
| **Bigtable** | Exabytes | Eventual | 99.9% |
| **Cosmos DB** | Global | Tunable | 99.999% |
| **MySQL** | Terabytes | Strong | 99.9% |
| **Cassandra** | Petabytes | Tunable | 99.99% |
| **Redis** | Hundreds of GB | Strong | 99.9% |
| **ClickHouse** | Petabytes | Eventual | 99.9% |

## Key Takeaways

### Google's Approach
- **Custom-built** for unprecedented scale
- **Strong consistency** even at global scale (Spanner)
- **Specialized systems** for different workloads

### Microsoft's Approach  
- **Enterprise-grade** features and compliance
- **Multi-model** flexibility (Cosmos DB)
- **Hybrid cloud** integration

### Uber's Approach
- **Polyglot persistence** - right tool for each job
- **Real-time performance** for operational systems
- **Open-source** technologies with custom optimizations

Each company's database choices reflect their core business needs: Google prioritizes global scale, Microsoft focuses on enterprise features, and Uber optimizes for real-time operations.
