# Module 28: Storage & File Systems

> Every system needs to store data — from user uploads and application logs to database files and machine learning models. Choosing the right storage type, understanding distributed file systems, designing for object storage, and selecting the right serialization format are decisions that affect performance, cost, scalability, and reliability. This module covers the storage landscape from block storage to object storage, distributed file systems like HDFS, data serialization formats, and compression strategies.

> **Ruby Context:** Ruby applications interact with all three storage types daily. **Block storage** backs your PostgreSQL and Redis instances (EBS volumes). **File storage** provides shared access for assets across app servers (EFS/NFS). **Object storage** (S3) is the workhorse for user uploads, backups, and static assets. Key Ruby tools include the **aws-sdk-s3** gem (S3 operations), **ActiveStorage** (Rails file attachment framework), **Shrine** / **CarrierWave** (file upload libraries), **Oj** / **multi_json** (fast JSON), **msgpack** (MessagePack), **google-protobuf** (Protocol Buffers), **avro** gem (Apache Avro), and **zstd-ruby** / **snappy** gems for compression.

---

## 28.1 Storage Types

> There are three fundamental storage types in computing: **block storage**, **file storage**, and **object storage**. Each has different access patterns, performance characteristics, and use cases. Understanding the differences is essential for making the right storage choice in system design.

---

### Block Storage

**Block storage** divides data into fixed-size **blocks** (typically 512 bytes to 4 KB) and stores them on a raw storage device. The storage device has no concept of files or directories — it only knows about blocks identified by addresses. A file system (ext4, XFS, NTFS) is layered on top to organize blocks into files.

```
Block Storage:
  +-------+-------+-------+-------+-------+-------+
  |Block 0|Block 1|Block 2|Block 3|Block 4|Block 5|
  +-------+-------+-------+-------+-------+-------+
  
  The OS/file system maps files to blocks:
    /var/data/file.txt → Blocks 0, 1, 2
    /var/data/image.png → Blocks 3, 4, 5
  
  Applications see files; the storage device sees blocks.
```

**Characteristics:**

| Feature | Details |
|---------|---------|
| **Access pattern** | Random read/write at block level (like a hard drive) |
| **Performance** | Lowest latency, highest IOPS (direct block access) |
| **File system** | Required — OS formats the device with ext4, XFS, etc. |
| **Sharing** | Typically attached to ONE server at a time (not shared) |
| **Resizing** | Can expand (add blocks); shrinking is complex |
| **Use case** | Databases, OS boot volumes, high-performance applications |

**Cloud Block Storage:**

| Service | Provider | Key Features |
|---------|----------|-------------|
| **EBS (Elastic Block Store)** | AWS | Attached to EC2 instances; snapshots; multiple volume types (gp3, io2, st1) |
| **Persistent Disk** | GCP | Attached to Compute Engine VMs; regional replication |
| **Azure Managed Disks** | Azure | Attached to Azure VMs; Ultra, Premium, Standard tiers |

**EBS Volume Types (AWS):**

| Type | IOPS | Throughput | Latency | Use Case |
|------|------|-----------|---------|----------|
| **gp3** (General Purpose SSD) | 3,000-16,000 | 125-1,000 MB/s | ~1 ms | Default; most workloads |
| **io2 Block Express** (Provisioned IOPS) | Up to 256,000 | Up to 4,000 MB/s | Sub-ms | High-performance databases (Oracle, SAP) |
| **st1** (Throughput Optimized HDD) | 500 | 500 MB/s | ~5 ms | Big data, data warehouses, log processing |
| **sc1** (Cold HDD) | 250 | 250 MB/s | ~10 ms | Infrequently accessed data, archives |

> **Ruby Context:** Your Rails PostgreSQL database runs on block storage (typically EBS gp3). When you see slow database queries, the underlying EBS volume's IOPS limits might be the bottleneck. Redis also runs on block storage — for high-throughput Redis, io2 volumes provide sub-millisecond latency. You don't interact with block storage directly from Ruby code, but understanding it helps you diagnose performance issues.

**SAN (Storage Area Network):**
A dedicated high-speed network that connects servers to shared block storage devices. Used in enterprise data centers for high-performance, shared storage.

```
SAN Architecture:
  Server A ──┐
  Server B ──┼── SAN Switch ── Storage Array (disk shelves)
  Server C ──┘   (Fibre Channel    (RAID arrays, SSDs)
                  or iSCSI)
```

---

### File Storage

**File storage** organizes data into a hierarchical structure of **files and directories**, accessed via a file system protocol (NFS, SMB/CIFS). Multiple servers can access the same file system simultaneously.

```
File Storage:
  /shared
  ├── /shared/documents
  │   ├── report.pdf
  │   └── presentation.pptx
  ├── /shared/images
  │   ├── logo.png
  │   └── banner.jpg
  └── /shared/config
      └── app.yaml
  
  Multiple servers mount the same file system:
    Server A: mount /shared → reads/writes files
    Server B: mount /shared → reads/writes same files
    Server C: mount /shared → reads/writes same files
```

**Characteristics:**

| Feature | Details |
|---------|---------|
| **Access pattern** | Hierarchical (directories + files); POSIX file operations |
| **Sharing** | Multiple servers can access simultaneously (shared file system) |
| **Protocol** | NFS (Linux), SMB/CIFS (Windows), AFP (macOS) |
| **Locking** | File-level or byte-range locking for concurrent access |
| **Performance** | Moderate (network overhead for shared access) |
| **Use case** | Shared configuration, CMS, home directories, legacy applications |

**Cloud File Storage:**

| Service | Provider | Protocol | Key Features |
|---------|----------|----------|-------------|
| **EFS (Elastic File System)** | AWS | NFS v4 | Auto-scaling, serverless, multi-AZ |
| **Filestore** | GCP | NFS | Managed NFS for GKE and Compute Engine |
| **Azure Files** | Azure | SMB, NFS | Managed file shares, Azure AD integration |
| **FSx for Lustre** | AWS | Lustre | High-performance parallel file system for HPC/ML |

> **Ruby Context:** File storage (EFS/NFS) is useful when multiple Rails app servers need to share files — for example, if you're using `ActiveStorage` with the `:local` disk service and running multiple Puma processes on different servers. However, the modern approach is to use object storage (S3) instead, which scales better and doesn't require shared mounts. EFS is still useful for Kubernetes pods that need shared persistent volumes.

---

### Object Storage

**Object storage** stores data as **objects** — each object consists of the data itself, metadata (key-value pairs), and a unique identifier (key). Objects are stored in a flat namespace (no directories), organized into **buckets** (containers).

```
Object Storage:
  Bucket: "my-app-assets"
    ├── Key: "images/logo.png"        → Object (binary data + metadata)
    ├── Key: "images/banner.jpg"      → Object (binary data + metadata)
    ├── Key: "videos/intro.mp4"       → Object (binary data + metadata)
    ├── Key: "backups/db-2025-03.sql" → Object (binary data + metadata)
    └── Key: "logs/2025/03/15/app.log"→ Object (binary data + metadata)
  
  Note: "images/" is NOT a directory — it's part of the key name.
  The "/" is just a convention for organizing keys (prefix-based listing).
```

**Characteristics:**

| Feature | Details |
|---------|---------|
| **Access pattern** | HTTP API (PUT, GET, DELETE by key); no random byte-level access |
| **Namespace** | Flat (no real directories); prefixes simulate hierarchy |
| **Scalability** | Virtually unlimited (exabytes of data, billions of objects) |
| **Durability** | Extremely high (S3: 99.999999999% — 11 nines) |
| **Availability** | High (S3: 99.99%) |
| **Performance** | Higher latency than block/file (HTTP overhead); high throughput for large objects |
| **Cost** | Cheapest storage option (especially for infrequent access) |
| **Immutability** | Objects are typically immutable (overwrite = new version) |
| **Use case** | Static assets, backups, data lakes, media files, logs |

**Cloud Object Storage:**

| Service | Provider | Durability | Storage Classes |
|---------|----------|-----------|----------------|
| **S3** | AWS | 11 nines | Standard, Intelligent-Tiering, Glacier, Deep Archive |
| **GCS** | GCP | 11 nines | Standard, Nearline, Coldline, Archive |
| **Azure Blob** | Azure | 11+ nines | Hot, Cool, Cold, Archive |

> **Ruby Context:** Object storage (S3) is the default storage backend for Rails applications. **ActiveStorage**, **Shrine**, and **CarrierWave** all support S3 as a primary backend. The `aws-sdk-s3` gem provides the low-level client.

```ruby
# ActiveStorage configuration for S3 (config/storage.yml)
# amazon:
#   service: S3
#   access_key_id: <%= ENV['AWS_ACCESS_KEY_ID'] %>
#   secret_access_key: <%= ENV['AWS_SECRET_ACCESS_KEY'] %>
#   region: us-east-1
#   bucket: my-app-uploads

# Using ActiveStorage in a model
class User < ApplicationRecord
  has_one_attached :avatar        # Single file
  has_many_attached :documents    # Multiple files
end

# Upload
user.avatar.attach(io: File.open('/path/to/photo.jpg'),
                   filename: 'photo.jpg',
                   content_type: 'image/jpeg')

# Generate a URL (pre-signed for private buckets)
url = user.avatar.url(expires_in: 1.hour)

# Direct upload from browser (bypasses app server)
# ActiveStorage provides JavaScript for direct-to-S3 uploads
```

---

### When to Use Which

| Criteria | Block Storage | File Storage | Object Storage |
|----------|-------------|-------------|---------------|
| **Access pattern** | Random read/write (byte-level) | Hierarchical files (POSIX) | HTTP API (key-based) |
| **Performance** | Highest IOPS, lowest latency | Moderate | Higher latency, high throughput |
| **Sharing** | Single server (typically) | Multiple servers | Any number of clients (HTTP) |
| **Scalability** | Limited (per-volume) | Moderate | Virtually unlimited |
| **Cost** | Most expensive | Moderate | Cheapest |
| **Durability** | Depends on RAID/replication | Depends on implementation | Extremely high (11 nines) |
| **Best for** | Databases, OS volumes | Shared files, CMS, config | Static assets, backups, data lakes |

**Decision Guide:**

| Need | Use |
|------|-----|
| Database storage (PostgreSQL, MySQL) | Block storage (EBS gp3 or io2) |
| High-performance computing (HPC) | Block storage (io2) or parallel file system (FSx Lustre) |
| Shared configuration files across servers | File storage (EFS, NFS) |
| Kubernetes persistent volumes | Block storage (EBS CSI) or file storage (EFS CSI) |
| User-uploaded images, videos, documents | Object storage (S3) |
| Static website assets (CSS, JS, images) | Object storage (S3) + CDN (CloudFront) |
| Database backups | Object storage (S3 Glacier) |
| Data lake / analytics | Object storage (S3) + query engine (Athena, Spark) |
| Log storage and archival | Object storage (S3 Intelligent-Tiering) |
| Machine learning training data | Object storage (S3) |

---


## 28.2 Distributed File Systems

> A **distributed file system** stores files across multiple machines, providing a unified view of the data as if it were on a single file system. Distributed file systems are designed for storing and processing **very large datasets** (terabytes to petabytes) across clusters of commodity hardware.

---

### HDFS (Hadoop Distributed File System)

**HDFS** is the storage layer of the Hadoop ecosystem. It's designed for storing very large files (gigabytes to terabytes) with high throughput access, optimized for batch processing rather than low-latency access.

**Architecture:**

```
HDFS Architecture:

  +------------------+
  | NameNode         |  (Master — single node)
  | - File metadata  |  - Maps files to blocks
  | - Block locations |  - Tracks which DataNodes hold which blocks
  | - Namespace (dirs)|  - Handles file open/close/rename
  +--------+---------+
           |
  +--------v---------+  +------------------+  +------------------+
  | DataNode 1       |  | DataNode 2       |  | DataNode 3       |
  | Block A1         |  | Block A2 (replica)|  | Block A3 (replica)|
  | Block B1         |  | Block B2 (replica)|  | Block C1         |
  | Block C2 (replica)|  | Block C3 (replica)|  | Block B3 (replica)|
  +------------------+  +------------------+  +------------------+
  
  File "data.csv" (300 MB):
    Block A (128 MB) → stored on DataNode 1, replicated to DN2 and DN3
    Block B (128 MB) → stored on DataNode 1, replicated to DN2 and DN3
    Block C (44 MB)  → stored on DataNode 3, replicated to DN1 and DN2
```

**Key Concepts:**

| Concept | Details |
|---------|---------|
| **Block size** | 128 MB (default); much larger than OS block size (4 KB) to minimize NameNode metadata and maximize throughput |
| **Replication factor** | 3 (default); each block is stored on 3 different DataNodes |
| **Rack awareness** | Replicas are placed on different racks for fault tolerance (1 on local rack, 2 on a different rack) |
| **Write-once** | Files are append-only; no random writes (optimized for batch processing) |
| **NameNode** | Single master that stores all metadata in memory; SPOF (mitigated by Secondary NameNode or HA NameNode) |
| **DataNode** | Worker nodes that store actual data blocks; send heartbeats to NameNode |

**HDFS Read Flow:**

```
1. Client → NameNode: "I want to read /data/file.csv"
2. NameNode → Client: "Block A is on DN1, DN2, DN3; Block B is on DN1, DN2, DN3; ..."
3. Client → DN1: "Give me Block A" (reads from nearest DataNode)
4. Client → DN2: "Give me Block B" (reads from nearest DataNode)
5. Client assembles blocks into the complete file
```

**HDFS Write Flow:**

```
1. Client → NameNode: "I want to write /data/newfile.csv"
2. NameNode → Client: "Write Block A to DN1, DN3, DN5" (pipeline)
3. Client → DN1: sends Block A
4. DN1 → DN3: replicates Block A (pipeline replication)
5. DN3 → DN5: replicates Block A
6. DN5 → DN3 → DN1 → Client: ACK (all replicas written)
7. Client → NameNode: "Block A written successfully"
8. Repeat for remaining blocks
```

**HDFS Limitations:**
- **Not suitable for low-latency access** — designed for batch processing (high throughput, not low latency)
- **Small files problem** — each file/block consumes NameNode memory (~150 bytes per block); millions of small files exhaust NameNode memory
- **Single writer** — no concurrent writes to the same file
- **No random writes** — append-only
- **NameNode is a SPOF** — mitigated by HA NameNode (active-standby with shared edit log)

> **Ruby Context:** Ruby applications rarely interact with HDFS directly. Instead, Ruby services write data to S3 or Kafka, and Hadoop/Spark jobs read from those sources. If you need to interact with HDFS from Ruby, the `webhdfs` gem provides a REST API client for HDFS's WebHDFS interface.

```ruby
# WebHDFS client for Ruby (webhdfs gem)
require 'webhdfs'

client = WebHDFS::Client.new('namenode-host', 50070, 'hadoop-user')

# Write a file to HDFS
client.create('/data/exports/users.csv', File.read('users.csv'))

# Read a file from HDFS
content = client.read('/data/exports/users.csv')

# List directory
files = client.list('/data/exports/')
files.each { |f| puts "#{f['pathSuffix']} (#{f['length']} bytes)" }

# In practice, Ruby apps write to S3 and Spark reads from S3:
# Ruby → S3 → Spark job → S3 → Ruby reads results
```

---

### GFS (Google File System) — Paper Concepts

**GFS** (2003) is Google's proprietary distributed file system that inspired HDFS. The GFS paper introduced many concepts that became standard in distributed storage.

**Key GFS Concepts:**

| Concept | GFS | HDFS Equivalent |
|---------|-----|----------------|
| Master | Single master stores metadata | NameNode |
| Chunk Server | Stores data chunks | DataNode |
| Chunk size | 64 MB | 128 MB (HDFS default) |
| Replication | 3 replicas per chunk | 3 replicas per block |
| Consistency | Relaxed (defined, consistent, inconsistent regions) | Strong (single writer, pipeline replication) |
| Append | Record append (atomic, at-least-once) | Append-only writes |
| Lease | Master grants leases to chunk servers for mutations | NameNode grants leases for writes |

**GFS Design Assumptions (still relevant today):**
1. Component failures are the **norm**, not the exception (commodity hardware fails frequently)
2. Files are **huge** (multi-GB); optimize for large files, not small ones
3. Workloads are **append-heavy** (logs, crawl data); random writes are rare
4. **High sustained throughput** is more important than low latency
5. **Co-design** the file system with the applications that use it

---

### Ceph

**Ceph** is a unified distributed storage system that provides **block, file, and object storage** from a single platform.

```
Ceph Architecture:

  +------------------+  +------------------+  +------------------+
  | RBD              |  | CephFS           |  | RADOS Gateway    |
  | (Block Storage)  |  | (File Storage)   |  | (Object Storage) |
  | - VM disks       |  | - POSIX file sys |  | - S3/Swift API   |
  +--------+---------+  +--------+---------+  +--------+---------+
           |                     |                     |
  +--------v---------+-----------v-----------+---------v---------+
  |                    RADOS                                     |
  |  (Reliable Autonomic Distributed Object Store)               |
  |  - Distributed object storage layer                          |
  |  - CRUSH algorithm for data placement (no central lookup)    |
  |  - Self-healing, self-managing                               |
  +------------------+------------------+------------------+-----+
  | OSD 1            | OSD 2            | OSD 3            | ... |
  | (Object Storage  | (Object Storage  | (Object Storage  |     |
  |  Daemon — one    |  Daemon)         |  Daemon)         |     |
  |  per disk)       |                  |                  |     |
  +------------------+------------------+------------------+-----+
```

**Key Ceph Concepts:**

| Concept | Description |
|---------|-------------|
| **RADOS** | The core distributed object store; all storage types are built on top of it |
| **OSD (Object Storage Daemon)** | One per disk; stores objects, handles replication, recovery |
| **Monitor (MON)** | Maintains cluster map (which OSDs are alive, data placement rules) |
| **CRUSH algorithm** | Deterministic data placement — no central lookup table; clients calculate where data lives |
| **Placement Groups (PGs)** | Logical grouping of objects; mapped to OSDs by CRUSH |
| **Self-healing** | Automatically re-replicates data when an OSD fails |

> **Ruby Context:** Ceph's RADOS Gateway provides an S3-compatible API, so your Ruby application can use the same `aws-sdk-s3` gem to talk to Ceph as it would to AWS S3. This is common in on-premises or hybrid deployments where you want S3 compatibility without AWS.

```ruby
# Using aws-sdk-s3 with Ceph RADOS Gateway (S3-compatible)
require 'aws-sdk-s3'

s3 = Aws::S3::Client.new(
  endpoint: 'http://ceph-rgw.internal:7480',  # Ceph RADOS Gateway
  access_key_id: ENV['CEPH_ACCESS_KEY'],
  secret_access_key: ENV['CEPH_SECRET_KEY'],
  region: 'us-east-1',                         # Required but ignored by Ceph
  force_path_style: true                        # Required for Ceph
)

# Same API as AWS S3
s3.put_object(bucket: 'my-bucket', key: 'data/file.txt', body: 'Hello Ceph!')
obj = s3.get_object(bucket: 'my-bucket', key: 'data/file.txt')
puts obj.body.read  # => "Hello Ceph!"
```

**Ceph vs HDFS:**

| Feature | Ceph | HDFS |
|---------|------|------|
| Storage types | Block + File + Object (unified) | File only |
| Metadata | Distributed (no single master) | Centralized (NameNode — SPOF) |
| Data placement | CRUSH algorithm (computed, no lookup) | NameNode (centralized lookup) |
| Random writes | Supported | Not supported (append-only) |
| Small files | Handles well | Poor (NameNode memory per file) |
| Use case | General-purpose distributed storage | Big data batch processing (Hadoop) |
| Kubernetes | Native (Rook-Ceph) | Not common |

---


## 28.3 Object Storage

> **Object storage** (exemplified by Amazon S3) has become the default storage layer for modern applications. It stores unstructured data as objects in a flat namespace, accessed via HTTP APIs. Its virtually unlimited scalability, extreme durability (11 nines), and low cost make it the go-to choice for static assets, backups, data lakes, and media files.

---

### S3-Like Architecture

```
S3 Architecture (simplified):

  Client → HTTPS → S3 API Endpoint
                      |
              +-------v--------+
              | Request Router |  (routes to correct partition based on bucket + key)
              +-------+--------+
                      |
         +------------+------------+
         |            |            |
  +------v------+ +---v------+ +--v-------+
  | Partition 1 | |Partition 2| |Partition 3|
  | (key range  | |(key range | |(key range |
  |  A-H)       | | I-P)      | | Q-Z)      |
  +------+------+ +---+------+ +--+-------+
         |            |            |
  +------v------+ +---v------+ +--v-------+
  | Storage     | | Storage  | | Storage  |
  | Nodes       | | Nodes    | | Nodes    |
  | (3+ replicas| | (3+)     | | (3+)     |
  | across AZs) | |          | |          |
  +-------------+ +----------+ +----------+
```

**How S3 Achieves 11 Nines Durability:**
- Each object is stored across **multiple devices** in **multiple Availability Zones**
- Uses **erasure coding** (not just replication) — splits data into fragments with parity, so the object can be reconstructed even if some fragments are lost
- Continuous **integrity checking** — checksums verify data hasn't been corrupted
- Automatic **repair** — if a fragment is lost (disk failure), it's automatically reconstructed from remaining fragments
- **Geographic redundancy** — data is stored across physically separated facilities

```
Erasure Coding (simplified):
  Object "photo.jpg" (10 MB)
  Split into 6 data fragments + 4 parity fragments = 10 fragments
  Stored across 10 different devices in 3 AZs
  
  Can lose ANY 4 fragments and still reconstruct the original object!
  (Only need 6 of 10 fragments)
  
  Storage overhead: 10/6 = 1.67x (vs 3x for triple replication)
  Durability: much higher than triple replication
```

---

### Buckets, Objects, Keys

**Bucket:** A container for objects. Globally unique name within a cloud provider.

```
Bucket: "my-company-assets"
  Region: us-east-1
  Settings: versioning enabled, encryption enabled, public access blocked
```

**Object:** The data stored in a bucket. Consists of:
- **Key:** The unique identifier (like a file path): `images/2025/03/photo.jpg`
- **Value:** The actual data (binary content, up to 5 TB per object in S3)
- **Metadata:** Key-value pairs (content-type, custom metadata, creation date)
- **Version ID:** If versioning is enabled, each version has a unique ID

> **Ruby Context:** The `aws-sdk-s3` gem is the standard way to interact with S3 from Ruby. Here's a comprehensive example covering common operations.

```ruby
# S3 operations with aws-sdk-s3 gem
require 'aws-sdk-s3'

s3 = Aws::S3::Client.new(region: 'us-east-1')

# Upload an object
s3.put_object(
  bucket: 'my-app-assets',
  key: 'images/2025/03/photo.jpg',
  body: File.open('/tmp/photo.jpg', 'rb'),
  content_type: 'image/jpeg',
  metadata: {
    'photographer' => 'Alice',
    'location' => 'San Francisco'
  },
  server_side_encryption: 'AES256'
)

# Download an object
response = s3.get_object(
  bucket: 'my-app-assets',
  key: 'images/2025/03/photo.jpg'
)
File.open('/tmp/downloaded.jpg', 'wb') { |f| f.write(response.body.read) }

# List objects with prefix (simulating directory listing)
response = s3.list_objects_v2(
  bucket: 'my-app-assets',
  prefix: 'images/2025/03/',
  max_keys: 100
)
response.contents.each do |obj|
  puts "#{obj.key} (#{obj.size} bytes, modified: #{obj.last_modified})"
end

# Delete an object
s3.delete_object(bucket: 'my-app-assets', key: 'images/2025/03/photo.jpg')

# Check if object exists
begin
  s3.head_object(bucket: 'my-app-assets', key: 'images/2025/03/photo.jpg')
  puts "Object exists"
rescue Aws::S3::Errors::NotFound
  puts "Object does not exist"
end
```

**Key Naming Best Practices:**
- Use prefixes to organize objects: `images/`, `videos/`, `backups/`
- Include dates for time-series data: `logs/2025/03/15/app.log`
- Avoid sequential prefixes for high-throughput workloads (S3 partitions by key prefix — sequential prefixes can create hotspots). S3 has largely mitigated this since 2018, but it's still good practice.

---

### Versioning

**Object versioning** keeps every version of an object. When you overwrite or delete an object, the previous version is preserved.

```
Without Versioning:
  PUT photo.jpg (v1) → stored
  PUT photo.jpg (v2) → v1 is GONE forever
  DELETE photo.jpg   → object is GONE forever

With Versioning:
  PUT photo.jpg (v1) → stored as version "abc"
  PUT photo.jpg (v2) → stored as version "def"; v1 ("abc") still exists
  DELETE photo.jpg   → adds a "delete marker"; v1 and v2 still exist!
  
  GET photo.jpg      → 404 (delete marker)
  GET photo.jpg?versionId=abc → returns v1
  GET photo.jpg?versionId=def → returns v2
  DELETE delete marker → photo.jpg is "undeleted" (latest version is v2)
```

**Use Cases for Versioning:**
- **Accidental deletion protection** — recover deleted objects
- **Audit trail** — track all changes to an object
- **Rollback** — revert to a previous version
- **Compliance** — regulatory requirements to retain all versions

**Cost Implication:** Every version consumes storage. Use **lifecycle policies** to automatically delete old versions after a retention period.

---

### Lifecycle Policies

**Lifecycle policies** automatically transition objects between storage classes or delete them based on age or other criteria.

```
Lifecycle Policy Example:
  Rule 1: After 30 days → move to Infrequent Access (cheaper storage)
  Rule 2: After 90 days → move to Glacier (archive storage)
  Rule 3: After 365 days → delete permanently
  Rule 4: Delete non-current versions after 30 days
  Rule 5: Abort incomplete multipart uploads after 7 days
```

**S3 Storage Classes:**

| Class | Cost (per GB/month) | Access Latency | Min Duration | Use Case |
|-------|-------------------|---------------|-------------|----------|
| **Standard** | $0.023 | Milliseconds | None | Frequently accessed data |
| **Intelligent-Tiering** | $0.023 (auto-tiers) | Milliseconds | None | Unknown access patterns |
| **Standard-IA** | $0.0125 | Milliseconds | 30 days | Infrequently accessed, rapid access needed |
| **One Zone-IA** | $0.01 | Milliseconds | 30 days | Infrequent, non-critical (single AZ) |
| **Glacier Instant** | $0.004 | Milliseconds | 90 days | Archive with instant access |
| **Glacier Flexible** | $0.0036 | Minutes to hours | 90 days | Archive, retrieval in minutes-hours |
| **Glacier Deep Archive** | $0.00099 | 12-48 hours | 180 days | Long-term archive, rarely accessed |

> **Ruby Context:** You can configure lifecycle policies via the `aws-sdk-s3` gem or through Terraform/CloudFormation. For Rails apps, setting up lifecycle policies for user uploads is a common cost optimization.

```ruby
# Configure S3 lifecycle policy via aws-sdk-s3
s3 = Aws::S3::Client.new(region: 'us-east-1')

s3.put_bucket_lifecycle_configuration(
  bucket: 'my-app-uploads',
  lifecycle_configuration: {
    rules: [
      {
        id: 'transition-to-ia',
        status: 'Enabled',
        filter: { prefix: 'uploads/' },
        transitions: [
          { days: 30, storage_class: 'STANDARD_IA' },
          { days: 90, storage_class: 'GLACIER' }
        ]
      },
      {
        id: 'delete-old-versions',
        status: 'Enabled',
        filter: { prefix: '' },
        noncurrent_version_expiration: { noncurrent_days: 30 }
      },
      {
        id: 'abort-incomplete-uploads',
        status: 'Enabled',
        filter: { prefix: '' },
        abort_incomplete_multipart_upload: { days_after_initiation: 7 }
      }
    ]
  }
)
```

---

### Pre-Signed URLs

A **pre-signed URL** grants temporary access to a private S3 object without requiring the requester to have AWS credentials. The URL includes a signature and expiration time.

```
Use Case: User uploads a profile picture

Without Pre-Signed URL:
  Client → App Server → App Server uploads to S3
  (App server is a bottleneck — all data flows through it)

With Pre-Signed URL:
  1. Client → App Server: "I want to upload a photo"
  2. App Server generates a pre-signed PUT URL (valid for 15 minutes)
  3. App Server → Client: "Upload directly to this URL"
  4. Client → S3: PUT directly to the pre-signed URL (bypasses app server!)
  5. Client → App Server: "Upload complete, here's the key"
```

> **Ruby Context:** Pre-signed URLs are essential for Rails applications handling file uploads. ActiveStorage uses them for direct uploads. Here's how to generate them manually with `aws-sdk-s3`.

```ruby
# Generate pre-signed URLs with aws-sdk-s3
require 'aws-sdk-s3'

signer = Aws::S3::Presigner.new(client: Aws::S3::Client.new(region: 'us-east-1'))

# Pre-signed URL for DOWNLOAD (GET)
download_url = signer.presigned_url(
  :get_object,
  bucket: 'my-app-assets',
  key: 'images/photo.jpg',
  expires_in: 3600  # 1 hour
)
# => "https://my-app-assets.s3.amazonaws.com/images/photo.jpg?X-Amz-..."

# Pre-signed URL for UPLOAD (PUT)
upload_url = signer.presigned_url(
  :put_object,
  bucket: 'my-app-uploads',
  key: "uploads/#{SecureRandom.uuid}/photo.jpg",
  expires_in: 900,  # 15 minutes
  content_type: 'image/jpeg'
)

# Rails controller for direct upload
class UploadsController < ApplicationController
  def presign
    key = "uploads/#{current_user.id}/#{SecureRandom.uuid}/#{params[:filename]}"

    presigned = signer.presigned_url(
      :put_object,
      bucket: ENV['S3_UPLOAD_BUCKET'],
      key: key,
      expires_in: 900,
      content_type: params[:content_type],
      metadata: { 'user-id' => current_user.id.to_s }
    )

    render json: { url: presigned, key: key }
  end

  private

  def signer
    @signer ||= Aws::S3::Presigner.new(
      client: Aws::S3::Client.new(region: ENV['AWS_REGION'])
    )
  end
end

# ActiveStorage direct upload (built-in pre-signed URL support)
# In your form:
# <%= form.file_field :avatar, direct_upload: true %>
# ActiveStorage handles pre-signed URL generation automatically
```

**Benefits:**
- **Offload bandwidth** — large files go directly to/from S3, not through your servers
- **Security** — temporary access; URL expires after the specified time
- **Scalability** — S3 handles the upload/download, not your application servers
- **Cost** — reduces data transfer through your servers

---

### Multipart Upload

For large objects (> 100 MB), **multipart upload** splits the object into parts, uploads them in parallel, and assembles them on the server.

```
Multipart Upload Flow:
  1. Initiate: Client → S3: "Start multipart upload for video.mp4"
     S3 → Client: upload_id = "abc123"

  2. Upload Parts (in parallel):
     Client → S3: PUT part 1 (100 MB) → ETag: "etag1"
     Client → S3: PUT part 2 (100 MB) → ETag: "etag2"
     Client → S3: PUT part 3 (100 MB) → ETag: "etag3"
     Client → S3: PUT part 4 (50 MB)  → ETag: "etag4"

  3. Complete: Client → S3: "Complete upload abc123 with parts [1:etag1, 2:etag2, 3:etag3, 4:etag4]"
     S3 assembles the parts into the final object: video.mp4 (350 MB)
```

> **Ruby Context:** The `aws-sdk-s3` gem handles multipart uploads automatically for large files. You can also use the `Aws::S3::MultipartFileUploader` for explicit control.

```ruby
# Automatic multipart upload (aws-sdk-s3 handles it)
s3_resource = Aws::S3::Resource.new(region: 'us-east-1')
obj = s3_resource.bucket('my-app-assets').object('videos/large-video.mp4')

# Automatically uses multipart for files > 15 MB (configurable)
obj.upload_file('/tmp/large-video.mp4',
  content_type: 'video/mp4',
  multipart_threshold: 100 * 1024 * 1024  # Use multipart for > 100 MB
)

# Manual multipart upload for streaming data
s3 = Aws::S3::Client.new(region: 'us-east-1')

# 1. Initiate
multipart = s3.create_multipart_upload(
  bucket: 'my-app-assets',
  key: 'exports/large-report.csv',
  content_type: 'text/csv'
)

# 2. Upload parts
parts = []
part_number = 1
File.open('/tmp/large-report.csv', 'rb') do |file|
  while (chunk = file.read(100 * 1024 * 1024))  # 100 MB chunks
    resp = s3.upload_part(
      bucket: 'my-app-assets',
      key: 'exports/large-report.csv',
      upload_id: multipart.upload_id,
      part_number: part_number,
      body: chunk
    )
    parts << { etag: resp.etag, part_number: part_number }
    part_number += 1
  end
end

# 3. Complete
s3.complete_multipart_upload(
  bucket: 'my-app-assets',
  key: 'exports/large-report.csv',
  upload_id: multipart.upload_id,
  multipart_upload: { parts: parts }
)
```

**Multipart Upload Benefits:**

| Benefit | Description |
|---------|-------------|
| **Parallel uploads** | Upload multiple parts simultaneously → faster |
| **Retry individual parts** | If one part fails, retry just that part (not the entire file) |
| **Resume uploads** | If the connection drops, resume from the last completed part |
| **Large files** | Required for objects > 5 GB; recommended for > 100 MB |
| **Network efficiency** | Better utilization of available bandwidth |

**Part Size Limits:**
- Minimum part size: 5 MB (except the last part)
- Maximum part size: 5 GB
- Maximum parts: 10,000
- Maximum object size: 5 TB (10,000 × 5 GB)

---


## 28.4 Data Serialization

> **Serialization** is the process of converting data structures into a format that can be stored or transmitted and later reconstructed. The choice of serialization format affects payload size, parsing speed, schema evolution, and developer experience. In system design, you'll encounter serialization in API communication, message queues, database storage, and file formats.

---

### Format Comparison

| Format | Type | Human Readable | Schema | Size | Speed | Language Support |
|--------|------|---------------|--------|------|-------|-----------------|
| **JSON** | Text | ✅ Yes | Optional (JSON Schema) | Large | Moderate | Universal |
| **XML** | Text | ✅ Yes (verbose) | Optional (XSD) | Largest | Slow | Universal |
| **Protocol Buffers** | Binary | ❌ No | Required (.proto) | Small | Very fast | 10+ languages |
| **Apache Avro** | Binary | ❌ No | Required (JSON schema) | Small | Fast | Java, Python, C, Ruby |
| **Apache Thrift** | Binary | ❌ No | Required (.thrift) | Small | Fast | 20+ languages |
| **MessagePack** | Binary | ❌ No | Optional | Small | Fast | 50+ languages |

---

### JSON

**JSON (JavaScript Object Notation)** is the most widely used serialization format for web APIs and configuration files.

```json
{
  "id": 123,
  "name": "Alice Smith",
  "email": "alice@example.com",
  "age": 30,
  "is_active": true,
  "tags": ["premium", "early-adopter"],
  "address": {
    "street": "123 Main St",
    "city": "San Francisco"
  }
}
```

| Pros | Cons |
|------|------|
| Human-readable and writable | Verbose (field names repeated in every object) |
| Universal support (every language) | No native binary type (must base64-encode) |
| Self-describing (field names in data) | No schema enforcement (by default) |
| Easy to debug (curl, browser, Postman) | Slower parsing than binary formats |
| Native in JavaScript/browsers | No comments (in strict JSON) |
| Flexible schema | Larger payload than binary formats |

> **Ruby Context:** Ruby has multiple JSON libraries with different performance characteristics. The built-in `json` stdlib is adequate, but **Oj** (Optimized JSON) is 2-10x faster and the standard choice for performance-sensitive Rails applications.

```ruby
# Standard library JSON
require 'json'
data = { id: 123, name: 'Alice', tags: ['premium'] }
json_string = JSON.generate(data)       # Serialize
parsed = JSON.parse(json_string)         # Deserialize

# Oj — 2-10x faster JSON (oj gem)
require 'oj'
Oj.default_options = { mode: :compat }  # Rails-compatible mode
json_string = Oj.dump(data)             # Serialize (fast)
parsed = Oj.load(json_string)           # Deserialize (fast)

# Rails uses multi_json which auto-selects the fastest available backend
# Add 'oj' to your Gemfile and Rails will use it automatically

# JSON serialization in Rails API
class UsersController < ApplicationController
  def show
    user = User.find(params[:id])
    render json: UserSerializer.new(user)  # Uses Oj if available
  end
end

# Benchmark: Oj vs stdlib JSON
require 'benchmark'
data = { users: 1000.times.map { |i| { id: i, name: "User #{i}", email: "user#{i}@example.com" } } }

Benchmark.bm do |x|
  x.report('JSON.generate') { 1000.times { JSON.generate(data) } }
  x.report('Oj.dump')       { 1000.times { Oj.dump(data) } }
end
# Oj is typically 2-5x faster for serialization, 2-10x for deserialization
```

**Best for:** REST APIs, configuration files, logging, any human-facing data exchange.

---

### XML

**XML (eXtensible Markup Language)** is an older, more verbose format still used in enterprise systems, SOAP APIs, and document formats.

| Pros | Cons |
|------|------|
| Rich schema validation (XSD, DTD) | Very verbose (opening + closing tags) |
| Namespace support | Slow to parse |
| XSLT for transformation | Complex (attributes vs elements, namespaces) |
| Mature tooling | Larger payload than JSON |
| Document-oriented (mixed content) | Declining usage (replaced by JSON) |

> **Ruby Context:** Ruby has excellent XML support via **Nokogiri** (the most popular XML/HTML parser) and **Builder** (XML generation). SOAP APIs still use XML, and the `savon` gem handles SOAP communication.

```ruby
# XML parsing with Nokogiri
require 'nokogiri'

xml = '<user><id>123</id><name>Alice</name><tags><tag>premium</tag></tags></user>'
doc = Nokogiri::XML(xml)
name = doc.at_xpath('//name').text  # => "Alice"

# XML generation with Builder
require 'builder'

xml = Builder::XmlMarkup.new(indent: 2)
result = xml.user do
  xml.id(123)
  xml.name('Alice')
  xml.tags { xml.tag('premium') }
end

# SOAP API call with Savon
require 'savon'

client = Savon.client(wsdl: 'https://legacy-api.example.com/service?wsdl')
response = client.call(:get_user, message: { user_id: 123 })
response.body[:get_user_response][:user]
```

**Best for:** Legacy enterprise systems, SOAP APIs, document formats (DOCX, SVG), configuration (Maven pom.xml).

---

### Protocol Buffers (protobuf)

**Protocol Buffers** (Google) is a binary serialization format with a required schema (.proto file). Covered in Module 16.3 for gRPC.

```protobuf
// Schema definition (.proto file)
message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
  int32 age = 4;
  bool is_active = 5;
  repeated string tags = 6;
  Address address = 7;
}

message Address {
  string street = 1;
  string city = 2;
}
```

| Pros | Cons |
|------|------|
| 3-10x smaller than JSON | Not human-readable (binary) |
| 5-100x faster parsing | Requires schema (.proto file) and code generation |
| Strong typing with code generation | Harder to debug (need protobuf tools) |
| Excellent backward/forward compatibility | Not self-describing (need schema to decode) |
| Used by gRPC | Steeper learning curve |

> **Ruby Context:** The `google-protobuf` gem provides Protocol Buffer support for Ruby. You define `.proto` files and generate Ruby classes with `protoc`.

```ruby
# After generating Ruby classes from .proto file:
# protoc --ruby_out=lib/ user.proto

require 'google/protobuf'

# Define schema in Ruby (alternative to .proto file)
Google::Protobuf::DescriptorPool.generated_pool.build do
  add_message 'User' do
    optional :id, :int32, 1
    optional :name, :string, 2
    optional :email, :string, 3
    optional :age, :int32, 4
    optional :is_active, :bool, 5
    repeated :tags, :string, 6
  end
end

User = Google::Protobuf::DescriptorPool.generated_pool.lookup('User').msgclass

# Serialize
user = User.new(id: 123, name: 'Alice', email: 'alice@example.com', age: 30, is_active: true)
binary = User.encode(user)        # => binary string (~60 bytes vs ~200 for JSON)

# Deserialize
decoded = User.decode(binary)
puts decoded.name  # => "Alice"

# JSON interop (for debugging)
json = User.encode_json(user)     # => JSON string
from_json = User.decode_json(json)
```

**Best for:** gRPC microservice communication, high-performance data exchange, mobile apps (bandwidth-sensitive).

---

### Apache Avro

**Avro** is a binary serialization format designed for Hadoop and big data. Its key differentiator: the **schema is stored with the data**, making it self-describing.

| Pros | Cons |
|------|------|
| Schema stored with data (self-describing) | Less language support than protobuf |
| Excellent schema evolution (reader/writer schema resolution) | Slower than protobuf for small messages |
| Compact binary encoding | Primarily used in Java/Hadoop ecosystem |
| Native Hadoop/Spark support | Less common outside big data |
| Schema Registry integration (Confluent) | |

> **Ruby Context:** The `avro` gem provides Avro support for Ruby. It's most commonly used with Kafka and the Confluent Schema Registry via the `avro_turf` gem.

```ruby
# Avro serialization with avro gem
require 'avro'

# Define schema
schema = Avro::Schema.parse('{
  "type": "record",
  "name": "User",
  "fields": [
    {"name": "id", "type": "int"},
    {"name": "name", "type": "string"},
    {"name": "email", "type": "string"},
    {"name": "age", "type": ["null", "int"], "default": null}
  ]
}')

# Serialize to binary
buffer = StringIO.new
writer = Avro::IO::DatumWriter.new(schema)
encoder = Avro::IO::BinaryEncoder.new(buffer)
writer.write({ 'id' => 123, 'name' => 'Alice', 'email' => 'alice@example.com', 'age' => 30 }, encoder)
binary = buffer.string

# Deserialize from binary
buffer = StringIO.new(binary)
reader = Avro::IO::DatumReader.new(schema)
decoder = Avro::IO::BinaryDecoder.new(buffer)
record = reader.read(decoder)
# => {"id"=>123, "name"=>"Alice", "email"=>"alice@example.com", "age"=>30}

# Avro with Kafka Schema Registry (avro_turf gem)
require 'avro_turf/messaging'

avro = AvroTurf::Messaging.new(registry_url: ENV['SCHEMA_REGISTRY_URL'])

# Encode with schema registry (schema is registered and cached)
encoded = avro.encode({ 'id' => 123, 'name' => 'Alice' }, schema_name: 'User')

# Decode (schema is fetched from registry by ID embedded in the message)
decoded = avro.decode(encoded)
```

**Best for:** Kafka messages (with Schema Registry), Hadoop/Spark data files, data lake storage.

---

### MessagePack

**MessagePack** is a binary format that's like "binary JSON" — same data model as JSON but encoded in binary for smaller size and faster parsing.

```
JSON:        {"name":"Alice","age":30}     → 27 bytes
MessagePack: \x82\xa4name\xa5Alice\xa3age\x1e → 18 bytes (33% smaller)
```

| Pros | Cons |
|------|------|
| Drop-in replacement for JSON (same data model) | No schema (like JSON) |
| 50+ language support | Not as compact as protobuf (field names still included) |
| Faster than JSON parsing | Not as fast as protobuf |
| No schema needed | No code generation |

> **Ruby Context:** The `msgpack` gem provides MessagePack support for Ruby. It's a popular choice when you want smaller/faster serialization without the complexity of protobuf schemas. Redis uses MessagePack internally, and it's a good format for Sidekiq job arguments or Redis cached values.

```ruby
# MessagePack with msgpack gem
require 'msgpack'

data = { 'id' => 123, 'name' => 'Alice', 'tags' => ['premium', 'early-adopter'] }

# Serialize (33% smaller than JSON, 2-5x faster)
packed = MessagePack.pack(data)    # => binary string (compact)
packed.bytesize                     # => ~45 bytes (vs ~70 for JSON)

# Deserialize
unpacked = MessagePack.unpack(packed)
# => {"id"=>123, "name"=>"Alice", "tags"=>["premium", "early-adopter"]}

# Use with Redis for compact cached values
require 'redis'

redis = Redis.new
redis.set('user:123', MessagePack.pack(user_data))
user = MessagePack.unpack(redis.get('user:123'))

# Streaming packer/unpacker for large datasets
packer = MessagePack::Packer.new
records.each { |record| packer.write(record) }
packed_stream = packer.to_s

unpacker = MessagePack::Unpacker.new
unpacker.feed(packed_stream)
unpacker.each { |record| process(record) }
```

**Best for:** When you want smaller/faster JSON without the complexity of protobuf schemas. Redis uses MessagePack internally.

---

### Schema Evolution and Backward Compatibility

As systems evolve, data formats change. **Schema evolution** is the ability to change the schema while maintaining compatibility with old data and old code.

**Compatibility Types:**

| Type | Definition | Example |
|------|-----------|---------|
| **Backward compatible** | New code can read OLD data | Add a new optional field (old data doesn't have it → use default) |
| **Forward compatible** | Old code can read NEW data | Add a new field (old code ignores unknown fields) |
| **Full compatible** | Both backward and forward | Add optional fields with defaults; never remove or rename fields |

**Schema Evolution Rules (Protobuf/Avro):**

| Change | Safe? | Details |
|--------|-------|---------|
| Add optional field | ✅ | Old data uses default; old code ignores new field |
| Remove optional field | ✅ | Mark as `reserved`; old code ignores missing field |
| Rename a field | ✅ (protobuf) | Wire format uses field numbers, not names |
| Change field number | ❌ | Breaks all existing data |
| Change field type | ❌ | Binary encoding is type-dependent |
| Add required field | ❌ | Old data doesn't have it → deserialization fails |
| Remove required field | ❌ | Old code expects it → deserialization fails |

> **Ruby Context:** Schema evolution matters most when using Kafka with Avro (via `avro_turf` + Schema Registry) or when storing protobuf in databases/caches. For JSON APIs, Rails serializers (e.g., `jsonapi-serializer`, `blueprinter`) handle versioning at the application level — you add new fields and old clients simply ignore them.

```ruby
# JSON API versioning in Rails (backward compatible)
class UserSerializer
  def initialize(user, version: :v2)
    @user = user
    @version = version
  end

  def as_json
    base = { id: @user.id, name: @user.name, email: @user.email }

    # v2 adds new fields — v1 clients ignore them (forward compatible)
    if @version >= :v2
      base[:avatar_url] = @user.avatar_url
      base[:created_at] = @user.created_at.iso8601
    end

    base
  end
end

# Protobuf schema evolution — add optional fields safely
# user_v2.proto:
# message User {
#   int32 id = 1;
#   string name = 2;
#   string email = 3;
#   string avatar_url = 4;  // NEW — old code ignores this field
#   // reserved 5;           // Field 5 was removed — reserve the number
# }
```

**Schema Registry (Confluent):**
For Kafka-based systems, a **Schema Registry** stores and manages Avro/protobuf/JSON schemas, enforcing compatibility rules on schema changes.

```
Producer → Schema Registry: "Register schema v2 for topic 'users'"
Schema Registry: checks compatibility with v1 → COMPATIBLE → registered
Producer → Kafka: send message with schema ID (not the full schema)
Consumer → Schema Registry: "Give me schema for ID 42"
Consumer: deserializes message using the schema
```

---


## 28.5 Data Compression

> **Compression** reduces the size of data for storage and transmission. In system design, compression affects storage costs, network bandwidth, latency, and CPU usage. The right compression algorithm depends on whether you prioritize compression ratio (smaller size) or speed (faster compression/decompression).

---

### Compression Algorithms Comparison

| Algorithm | Compression Ratio | Compression Speed | Decompression Speed | CPU Usage | Use Case |
|-----------|------------------|------------------|-------------------|-----------|----------|
| **gzip** | Good (60-80% reduction) | Moderate | Moderate | Moderate | HTTP responses, file archives |
| **Brotli** | Better (10-25% better than gzip) | Slow (compression) | Fast (decompression) | High (compression) | Static assets (pre-compressed), HTTP |
| **snappy** | Moderate (40-60%) | Very fast | Very fast | Low | Real-time: Kafka, database pages, RPC |
| **lz4** | Moderate (40-60%) | Fastest | Fastest | Lowest | Real-time: database pages, IPC, gaming |
| **zstd** | Excellent (close to gzip, faster) | Fast | Very fast | Moderate | General purpose: logs, backups, databases |

> **Ruby Context:** Ruby has gems for all major compression algorithms. The most commonly used in Rails applications are **gzip** (built-in via `Zlib`), **brotli** (`brotli` gem), **snappy** (`snappy` gem), **zstd** (`zstd-ruby` gem), and **lz4** (`lz4-ruby` gem). Rails middleware handles HTTP compression automatically.

```ruby
# gzip — built into Ruby stdlib
require 'zlib'

data = "Hello " * 10_000  # 60,000 bytes

# Compress
compressed = Zlib::Deflate.deflate(data)
compressed.bytesize  # => ~100 bytes (99.8% reduction for repetitive data)

# Decompress
decompressed = Zlib::Inflate.inflate(compressed)

# gzip format (for files and HTTP)
gz_data = ActiveSupport::Gzip.compress(data)    # Rails helper
original = ActiveSupport::Gzip.decompress(gz_data)

# Snappy — very fast, moderate compression (snappy gem)
require 'snappy'

compressed = Snappy.deflate(data)
decompressed = Snappy.inflate(compressed)

# Zstandard — best general-purpose (zstd-ruby gem)
require 'zstd-ruby'

compressed = Zstd.compress(data, level: 3)  # Levels 1-22
decompressed = Zstd.decompress(compressed)

# Benchmark comparison
require 'benchmark'

Benchmark.bm(15) do |x|
  x.report('gzip compress')   { 100.times { Zlib::Deflate.deflate(data) } }
  x.report('snappy compress') { 100.times { Snappy.deflate(data) } }
  x.report('zstd compress')   { 100.times { Zstd.compress(data) } }
  x.report('gzip decompress') { 100.times { Zlib::Inflate.inflate(Zlib::Deflate.deflate(data)) } }
  x.report('snappy decomp')   { 100.times { Snappy.inflate(Snappy.deflate(data)) } }
  x.report('zstd decompress') { 100.times { Zstd.decompress(Zstd.compress(data)) } }
end
```

---

### Compression vs Speed Trade-offs

```
Compression Ratio vs Speed:

  Ratio (smaller)
    ^
    |  * Brotli (level 11)
    |
    |  * zstd (level 19)
    |  * gzip (level 9)
    |
    |  * zstd (level 3)
    |  * gzip (level 1)
    |
    |  * snappy
    |  * lz4
    |
    +──────────────────────────────→ Speed (faster)
    
  Top-left: best compression, slowest
  Bottom-right: worst compression, fastest
  zstd offers the best balance (good compression + good speed)
```

**Benchmark (compressing 1 GB of JSON log data):**

| Algorithm | Compressed Size | Compression Time | Decompression Time | Ratio |
|-----------|----------------|-----------------|-------------------|-------|
| None | 1000 MB | 0s | 0s | 1.0x |
| lz4 | 480 MB | 0.8s | 0.3s | 2.1x |
| snappy | 500 MB | 0.9s | 0.4s | 2.0x |
| zstd (level 1) | 350 MB | 1.2s | 0.5s | 2.9x |
| zstd (level 3) | 300 MB | 2.0s | 0.5s | 3.3x |
| gzip (level 6) | 280 MB | 8.0s | 1.5s | 3.6x |
| zstd (level 19) | 250 MB | 30.0s | 0.5s | 4.0x |
| Brotli (level 11) | 230 MB | 60.0s | 1.0s | 4.3x |

*Note: These are approximate benchmarks. Actual results vary by data type and hardware.*

---

### When to Compress

| Scenario | Compress? | Algorithm | Why |
|----------|----------|-----------|-----|
| **HTTP API responses** | ✅ Yes | gzip or zstd | Reduces transfer time; browsers decompress automatically |
| **Static assets (CSS, JS)** | ✅ Yes | Brotli (pre-compressed) | Compress once, serve many times; best ratio |
| **Kafka messages** | ✅ Yes | snappy, lz4, or zstd | Reduces broker storage and network; producer compresses, consumer decompresses |
| **Database pages** | ✅ Yes | lz4 or zstd | Reduces disk I/O; transparent to queries |
| **Log files** | ✅ Yes | gzip or zstd | Reduces storage cost significantly |
| **Backups** | ✅ Yes | zstd or gzip | Reduces storage cost and transfer time |
| **Data lake files (Parquet)** | ✅ Yes | snappy or zstd | Built into the file format; reduces storage and scan time |
| **Images (JPEG, PNG, WebP)** | ❌ No | Already compressed | Re-compressing adds CPU overhead with no benefit |
| **Videos (H.264, H.265)** | ❌ No | Already compressed | Same — already compressed |
| **Encrypted data** | ❌ No | Encryption removes patterns | Compression relies on patterns; encrypted data has none |
| **Very small payloads (< 1 KB)** | ❌ Usually no | Overhead exceeds savings | Compression header + dictionary may be larger than savings |
| **Real-time, latency-critical** | ⚠️ Maybe | lz4 (fastest) | Only if the bandwidth savings outweigh the CPU cost |

> **Ruby Context:** Rails handles HTTP compression automatically via `Rack::Deflater` middleware (gzip). For Nginx in front of Rails, configure gzip and Brotli at the Nginx level for better performance. For cached data in Redis, compressing large values with snappy or zstd before storing can significantly reduce memory usage.

```ruby
# Enable gzip compression in Rails (config/application.rb)
config.middleware.use Rack::Deflater

# Nginx gzip configuration (recommended over Rack::Deflater)
# gzip on;
# gzip_types text/plain application/json application/javascript text/css;
# gzip_min_length 1000;
# gzip_comp_level 6;

# Compress cached values in Redis
class CompressedCache
  def initialize(redis: Redis.current, algorithm: :zstd)
    @redis = redis
    @algorithm = algorithm
  end

  def write(key, value, expires_in: nil)
    serialized = Oj.dump(value)

    compressed = case @algorithm
                 when :zstd   then Zstd.compress(serialized)
                 when :snappy then Snappy.deflate(serialized)
                 when :gzip   then Zlib::Deflate.deflate(serialized)
                 else serialized
                 end

    if expires_in
      @redis.setex(key, expires_in.to_i, compressed)
    else
      @redis.set(key, compressed)
    end
  end

  def read(key)
    compressed = @redis.get(key)
    return nil unless compressed

    decompressed = case @algorithm
                   when :zstd   then Zstd.decompress(compressed)
                   when :snappy then Snappy.inflate(compressed)
                   when :gzip   then Zlib::Inflate.inflate(compressed)
                   else compressed
                   end

    Oj.load(decompressed)
  end
end

# Usage — saves ~60-80% Redis memory for large JSON values
cache = CompressedCache.new(algorithm: :zstd)
cache.write('user:123:feed', large_feed_data, expires_in: 1.hour)
feed = cache.read('user:123:feed')

# Compress S3 uploads
require 'zlib'

# Compress log data before uploading to S3
log_data = File.read('/var/log/app/production.log')
compressed = Zlib::Deflate.deflate(log_data)

s3.put_object(
  bucket: 'my-app-logs',
  key: "logs/#{Date.today}/production.log.gz",
  body: compressed,
  content_encoding: 'gzip',
  content_type: 'text/plain'
)
```

**Compression in the Full Stack:**

```
Client → (Brotli/gzip compressed HTTP) → CDN
CDN → (Brotli/gzip) → Load Balancer (Nginx)
Nginx → (uncompressed) → Puma (Rails)
Rails → (snappy/zstd compressed) → Kafka (via Karafka)
Kafka → (snappy/zstd) → Consumer
Consumer → (zstd compressed) → S3 (Parquet files)
Rails → (zstd compressed values) → Redis
Rails → (uncompressed SQL) → PostgreSQL (lz4 compressed pages on disk)
```

---

## Summary & Key Takeaways

| Topic | Key Point |
|-------|-----------|
| Block Storage | Raw blocks; highest performance; databases and OS volumes; EBS, Persistent Disk |
| File Storage | Hierarchical files; shared access (NFS); EFS, Azure Files |
| Object Storage | Flat namespace (key → object); unlimited scale; 11 nines durability; S3, GCS |
| When to Use Which | Block for databases, file for shared access, object for everything else (default) |
| HDFS | Hadoop's distributed file system; 128 MB blocks; 3x replication; NameNode + DataNodes |
| HDFS Limitations | Not for low-latency; small files problem; NameNode SPOF; append-only |
| GFS | Google's DFS that inspired HDFS; 64 MB chunks; relaxed consistency |
| Ceph | Unified storage (block + file + object); CRUSH algorithm; no central metadata server |
| S3 Durability | 11 nines (99.999999999%); erasure coding across AZs |
| Versioning | Keep all versions of objects; protect against accidental deletion |
| Lifecycle Policies | Auto-transition between storage classes; Standard → IA → Glacier → Delete |
| Pre-Signed URLs | Temporary access to private objects; offload uploads/downloads from app servers |
| Multipart Upload | Parallel upload of large files; retry individual parts; required for > 5 GB |
| JSON | Human-readable; universal; verbose; best for REST APIs |
| Protobuf | Binary; 3-10x smaller than JSON; schema required; best for gRPC/microservices |
| Avro | Binary; schema with data; best for Kafka + Schema Registry |
| MessagePack | Binary JSON; no schema; drop-in replacement for JSON |
| Schema Evolution | Add optional fields (safe); never change field numbers/types (unsafe) |
| gzip | Good compression; moderate speed; HTTP default |
| Brotli | Best compression; slow to compress; pre-compress static assets |
| snappy/lz4 | Fastest; moderate compression; real-time use (Kafka, databases) |
| zstd | Best general-purpose; good compression + good speed; replacing gzip |
| Don't Compress | Already-compressed data (images, video), encrypted data, very small payloads |

**Ruby-Specific Takeaways:**

| Area | Ruby Tools & Gems |
|------|-------------------|
| **S3 / Object Storage** | `aws-sdk-s3`, ActiveStorage, Shrine, CarrierWave |
| **Pre-Signed URLs** | `Aws::S3::Presigner`, ActiveStorage direct uploads |
| **Multipart Upload** | `Aws::S3::Resource#upload_file` (automatic), manual via `aws-sdk-s3` |
| **HDFS** | `webhdfs` gem (WebHDFS REST client) |
| **Ceph** | `aws-sdk-s3` with custom endpoint (S3-compatible API) |
| **JSON (fast)** | `oj` gem (2-10x faster than stdlib), `multi_json` |
| **XML** | `nokogiri` (parsing), `builder` (generation), `savon` (SOAP) |
| **Protocol Buffers** | `google-protobuf` gem, `grpc` gem |
| **Avro** | `avro` gem, `avro_turf` (Schema Registry integration) |
| **MessagePack** | `msgpack` gem |
| **gzip** | `Zlib` (stdlib), `Rack::Deflater` (Rails middleware) |
| **Brotli** | `brotli` gem |
| **Snappy** | `snappy` gem |
| **Zstandard** | `zstd-ruby` gem |
| **LZ4** | `lz4-ruby` gem |

---

## Interview Tips for Module 28

1. **Block vs File vs Object** — know the differences; object storage (S3) is the default for most use cases; mention `aws-sdk-s3` and ActiveStorage for Ruby
2. **S3 durability** — explain 11 nines; mention erasure coding across AZs
3. **Pre-signed URLs** — explain how they offload uploads/downloads from app servers; draw the flow; show the Ruby `Aws::S3::Presigner` example
4. **Multipart upload** — explain parallel upload of parts; mention retry of individual parts; reference `upload_file` in `aws-sdk-s3`
5. **Storage class lifecycle** — explain Standard → IA → Glacier → Delete; mention cost savings
6. **HDFS architecture** — draw NameNode + DataNodes; explain block replication; mention the small files problem; note Ruby apps typically interact via S3, not HDFS directly
7. **Ceph vs HDFS** — Ceph is unified (block + file + object) with no single master; HDFS is file-only with NameNode SPOF; Ceph's S3-compatible API works with `aws-sdk-s3`
8. **JSON vs Protobuf** — JSON for human-readable APIs; protobuf for high-performance microservice communication; mention Oj for fast JSON in Ruby
9. **Avro vs Protobuf** — Avro stores schema with data (self-describing); protobuf needs schema at both ends; Avro for Kafka with `avro_turf` gem
10. **Schema evolution** — explain backward/forward compatibility; add optional fields (safe), never change field numbers (unsafe)
11. **Compression choice** — gzip for HTTP (Rack::Deflater or Nginx), Brotli for static assets, snappy/lz4 for real-time, zstd for general purpose; show compressed Redis cache pattern
12. **When NOT to compress** — already-compressed data (images, video), encrypted data, very small payloads