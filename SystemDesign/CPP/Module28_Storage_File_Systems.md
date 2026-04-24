# Module 28: Storage & File Systems

> Every system needs to store data — from user uploads and application logs to database files and machine learning models. Choosing the right storage type, understanding distributed file systems, designing for object storage, and selecting the right serialization format are decisions that affect performance, cost, scalability, and reliability. This module covers the storage landscape from block storage to object storage, distributed file systems like HDFS, data serialization formats, and compression strategies.

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

```
Object:
  Key:      "images/2025/03/photo.jpg"
  Value:    <binary data — 2.5 MB JPEG>
  Metadata:
    Content-Type: image/jpeg
    Content-Length: 2621440
    x-amz-meta-photographer: "Alice"
    x-amz-meta-location: "San Francisco"
    Last-Modified: 2025-03-15T10:30:00Z
    ETag: "d41d8cd98f00b204e9800998ecf8427e"
  Version ID: "v3_abc123"
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

Timeline for an object:
  Day 0:   Created in S3 Standard ($0.023/GB/month)
  Day 30:  Auto-moved to S3 IA ($0.0125/GB/month) — 46% cheaper
  Day 90:  Auto-moved to S3 Glacier ($0.004/GB/month) — 83% cheaper
  Day 365: Auto-deleted (no cost)
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

```
Pre-Signed URL for Download:
  https://my-bucket.s3.amazonaws.com/images/photo.jpg
    ?X-Amz-Algorithm=AWS4-HMAC-SHA256
    &X-Amz-Credential=AKIA.../20250315/us-east-1/s3/aws4_request
    &X-Amz-Date=20250315T100000Z
    &X-Amz-Expires=900          ← valid for 15 minutes
    &X-Amz-Signature=abc123...  ← cryptographic signature
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
     (Parts can be uploaded in parallel from multiple threads/machines!)

  3. Complete: Client → S3: "Complete upload abc123 with parts [1:etag1, 2:etag2, 3:etag3, 4:etag4]"
     S3 assembles the parts into the final object: video.mp4 (350 MB)
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
| **Apache Avro** | Binary | ❌ No | Required (JSON schema) | Small | Fast | Java, Python, C, C++ |
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

**Best for:** REST APIs, configuration files, logging, any human-facing data exchange.

---

### XML

**XML (eXtensible Markup Language)** is an older, more verbose format still used in enterprise systems, SOAP APIs, and document formats.

```xml
<user>
  <id>123</id>
  <name>Alice Smith</name>
  <email>alice@example.com</email>
  <tags>
    <tag>premium</tag>
    <tag>early-adopter</tag>
  </tags>
  <address>
    <street>123 Main St</street>
    <city>San Francisco</city>
  </address>
</user>
```

| Pros | Cons |
|------|------|
| Rich schema validation (XSD, DTD) | Very verbose (opening + closing tags) |
| Namespace support | Slow to parse |
| XSLT for transformation | Complex (attributes vs elements, namespaces) |
| Mature tooling | Larger payload than JSON |
| Document-oriented (mixed content) | Declining usage (replaced by JSON) |

**Best for:** Legacy enterprise systems, SOAP APIs, document formats (DOCX, SVG), configuration (Maven pom.xml, Android layouts).

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

**Binary Encoding (simplified):**
```
Field 1 (id=123):     [field_number=1, type=varint] [value=123]     → 3 bytes
Field 2 (name):       [field_number=2, type=string] [length=11] [Alice Smith] → 14 bytes
Field 3 (email):      [field_number=3, type=string] [length=17] [alice@...] → 20 bytes

Total: ~60 bytes (vs ~200 bytes for JSON equivalent)
```

| Pros | Cons |
|------|------|
| 3-10x smaller than JSON | Not human-readable (binary) |
| 5-100x faster parsing | Requires schema (.proto file) and code generation |
| Strong typing with code generation | Harder to debug (need protobuf tools) |
| Excellent backward/forward compatibility | Not self-describing (need schema to decode) |
| Used by gRPC | Steeper learning curve |

**Best for:** gRPC microservice communication, high-performance data exchange, mobile apps (bandwidth-sensitive).

---

### Apache Avro

**Avro** is a binary serialization format designed for Hadoop and big data. Its key differentiator: the **schema is stored with the data**, making it self-describing.

```json
// Avro schema (JSON format)
{
  "type": "record",
  "name": "User",
  "fields": [
    {"name": "id", "type": "int"},
    {"name": "name", "type": "string"},
    {"name": "email", "type": "string"},
    {"name": "age", "type": ["null", "int"], "default": null}
  ]
}
```

**Key Difference from Protobuf:**
- **Protobuf:** Schema is needed at both ends; field numbers in the binary data
- **Avro:** Schema is embedded in the file header; reader and writer schemas can differ (schema evolution via schema registry)

| Pros | Cons |
|------|------|
| Schema stored with data (self-describing) | Less language support than protobuf |
| Excellent schema evolution (reader/writer schema resolution) | Slower than protobuf for small messages |
| Compact binary encoding | Primarily used in Java/Hadoop ecosystem |
| Native Hadoop/Spark support | Less common outside big data |
| Schema Registry integration (Confluent) | |

**Best for:** Kafka messages (with Schema Registry), Hadoop/Spark data files, data lake storage.

---

### Apache Thrift

**Thrift** (Facebook, now Apache) is both a serialization format and an RPC framework, similar to protobuf + gRPC.

```thrift
// Thrift schema (.thrift file)
struct User {
  1: i32 id,
  2: string name,
  3: string email,
  4: optional i32 age,
  5: list<string> tags
}

service UserService {
  User getUser(1: i32 id),
  void createUser(1: User user)
}
```

| Pros | Cons |
|------|------|
| Serialization + RPC in one package | Less popular than protobuf/gRPC |
| 20+ language support | Smaller community |
| Multiple serialization protocols (binary, compact, JSON) | Less active development |

**Best for:** Legacy Facebook-ecosystem services. For new projects, protobuf + gRPC is generally preferred.

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

**Detailed Characteristics:**

**gzip:**
- The most widely supported compression format
- Used by default for HTTP response compression (`Content-Encoding: gzip`)
- Compression levels 1-9 (1=fastest, 9=best compression)
- Good balance of compression ratio and speed
- Supported by every browser, web server, and programming language

**Brotli:**
- Developed by Google; 10-25% better compression than gzip
- Compression is slow (not suitable for real-time compression)
- Decompression is fast (suitable for serving pre-compressed static assets)
- Supported by all modern browsers (`Content-Encoding: br`)
- Best used for static assets that are compressed once and served many times

**snappy:**
- Developed by Google; prioritizes speed over compression ratio
- Does not achieve the best compression but is extremely fast
- Used by Kafka (producer-side compression), LevelDB, Cassandra
- No compression levels — one speed, one ratio

**lz4:**
- The fastest compression algorithm available
- Slightly better compression than snappy, similar speed
- Used by ZFS, Linux kernel, Facebook's Zstandard as a fast mode
- Ideal when CPU is the bottleneck and you need minimal compression overhead

**zstd (Zstandard):**
- Developed by Facebook; the best general-purpose algorithm
- Compression ratio close to gzip but 3-5x faster
- Decompression is 2-3x faster than gzip
- Compression levels 1-22 (wide range of speed/ratio trade-offs)
- Dictionary compression for small messages (Kafka, HTTP)
- Rapidly replacing gzip in many use cases

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
    +──────────────────────────→ Speed (faster)
    
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

**Compression in the Full Stack:**

```
Client → (Brotli/gzip compressed HTTP) → CDN
CDN → (Brotli/gzip) → Load Balancer
Load Balancer → (uncompressed or gzip) → App Server
App Server → (snappy/lz4 compressed) → Kafka
Kafka → (snappy/lz4) → Consumer
Consumer → (zstd compressed) → S3 (Parquet files)
App Server → (lz4 compressed pages) → PostgreSQL
PostgreSQL → (lz4 compressed) → EBS (SSD)
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

---

## Interview Tips for Module 28

1. **Block vs File vs Object** — know the differences; object storage (S3) is the default for most use cases
2. **S3 durability** — explain 11 nines; mention erasure coding across AZs
3. **Pre-signed URLs** — explain how they offload uploads/downloads from app servers; draw the flow
4. **Multipart upload** — explain parallel upload of parts; mention retry of individual parts
5. **Storage class lifecycle** — explain Standard → IA → Glacier → Delete; mention cost savings
6. **HDFS architecture** — draw NameNode + DataNodes; explain block replication; mention the small files problem
7. **Ceph vs HDFS** — Ceph is unified (block + file + object) with no single master; HDFS is file-only with NameNode SPOF
8. **JSON vs Protobuf** — JSON for human-readable APIs; protobuf for high-performance microservice communication
9. **Avro vs Protobuf** — Avro stores schema with data (self-describing); protobuf needs schema at both ends; Avro for Kafka
10. **Schema evolution** — explain backward/forward compatibility; add optional fields (safe), never change field numbers (unsafe)
11. **Compression choice** — gzip for HTTP, Brotli for static assets, snappy/lz4 for real-time, zstd for general purpose
12. **When NOT to compress** — already-compressed data (images, video), encrypted data, very small payloads
